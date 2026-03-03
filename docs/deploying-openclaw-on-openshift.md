# Deploying OpenClaw on Red Hat OpenShift: A Practitioner's Guide

*This is Part 1 of the "OpenClaw on OpenShift" series. Future posts will cover network security with EgressFirewall and AdminNetworkPolicy, observability with the Cluster Logging Operator and ServiceMonitor, and scaling to multiple replicas with Gateway API.*

AI agent gateways are becoming a common workload for platform teams to support. [OpenClaw](https://openclaw.ai) is an open-source Node.js gateway that connects large language model (LLM) providers to messaging channels such as Telegram, Discord, Slack, Signal, and WhatsApp. It coordinates agents, tools, and sessions through a single process that serves both HTTP and WebSocket on one port.

Running OpenClaw on Red Hat OpenShift Container Platform gives you enterprise-grade TLS termination through Routes, security-by-default with the `restricted-v2` SecurityContextConstraint (SCC), network segmentation with OVN-Kubernetes, and a container build pipeline backed by Red Hat Universal Base Image (UBI). But deploying a stateful, filesystem-backed Node.js application on OpenShift involves some platform-specific decisions, particularly around storage, user identity, and networking, that are worth understanding up front.

In this post, we walk through deploying OpenClaw on OpenShift from scratch: building an OpenShift-native container image with UBI, creating each Kubernetes resource, and verifying the deployment end to end. By the end, you will have a running OpenClaw gateway with TLS and persistent storage, ready to connect to any supported messaging channel.

## Understanding the OpenClaw architecture

Before deploying, it helps to understand what we are putting on the cluster.

The OpenClaw gateway is a single-threaded Node.js process that multiplexes HTTP and WebSocket on port 18789. CLI and web clients connect over WebSocket. Messaging channels connect using their own native protocols: Telegram long-polls the Bot API, Discord maintains a WebSocket connection to Discord's gateway, Slack uses Socket Mode, and Signal uses Server-Sent Events. The gateway coordinates all of this in-process.

All state lives on the filesystem under a single configurable directory:

```
$OPENCLAW_STATE_DIR/
  openclaw.json                # Configuration (JSON5)
  identity/device.json         # ED25519 device keypair
  credentials/oauth.json       # OAuth tokens
  agents/<agentId>/
    sessions/
      sessions.json            # Session metadata
      <sessionId>.jsonl        # Session transcripts
    memory.db                  # SQLite memory index (sqlite-vec + FTS5)
```

There is no external database. Sessions are stored as JSONL files. The memory index uses SQLite through Node's built-in `node:sqlite` module with the `sqlite-vec` extension for vector search. Configuration is a single JSON5 file.

The gateway exposes two health endpoints on the same port, both of which require no authentication:

| Endpoint | Purpose |
|----------|---------|
| `GET /healthz` | Liveness probe |
| `GET /readyz` | Readiness probe |

## Target architecture on OpenShift

The following diagram shows the deployment topology we are building:

```
                    ┌──────────────────────────────┐
                    │       OpenShift Route         │
Internet ──HTTPS──▶ │  (edge TLS, port 443)        │
                    └──────────────┬───────────────┘
                                   │ HTTP (port 18789)
                    ┌──────────────▼───────────────┐
                    │          Service              │
                    │   ClusterIP, port 18789       │
                    └──────────────┬───────────────┘
                                   │
                    ┌──────────────▼───────────────┐
                    │            Pod                │
                    │  ┌────────────────────────┐   │
                    │  │   OpenClaw Gateway     │   │
                    │  │   node openclaw.mjs    │   │
                    │  │   :18789 (HTTP + WS)   │   │
                    │  └──────────┬─────────────┘   │
                    │             │                  │
                    │  ┌──────────▼─────────────┐   │
                    │  │   PVC (block storage)   │   │
                    │  │  /data/openclaw          │   │
                    │  └────────────────────────┘   │
                    └──────────────────────────────┘
                                   │
                     Outbound (HTTPS, WSS)
                                   ▼
                    Telegram API, Discord Gateway,
                    Anthropic API, OpenAI API, ...
```

Several aspects of this architecture are worth highlighting:

- **Single port.** HTTP health checks and WebSocket upgrades share port 18789. No sidecar or second port is needed for health probes.
- **Outbound-only channel traffic.** Most channels (Telegram in polling mode, Discord, Signal, WhatsApp) initiate outbound connections from the pod. The Route does not need to be internet-reachable for channel traffic. Telegram webhook mode and Slack HTTP mode are exceptions that require inbound connectivity.
- **Filesystem-backed state.** All persistent state lives on a single PVC. Backup reduces to a volume snapshot.
- **Single replica.** OpenClaw's session transcripts (JSONL) and memory index (SQLite) do not support concurrent writers, so this deployment uses a single replica. Horizontal scaling is addressed in a later post in this series.

## Prerequisites

To follow along, you will need:

- A Red Hat OpenShift Container Platform cluster with the `oc` CLI configured
- A block storage class, which is available by default on most platforms (`gp3-csi` on AWS, `managed-csi` on Azure, `thin-csi` on vSphere, or `lvms-vg1` on bare metal with the LVM Storage Operator)
- A model provider API key (for example, an Anthropic or OpenAI API key)
- Optionally, a messaging channel credential to test with. A Telegram bot token is the simplest choice (outbound-only polling, no webhook required). You can create one through [@BotFather](https://t.me/BotFather)
- Optionally, [Node.js 22+](https://nodejs.org/) on your workstation to install the OpenClaw CLI for verification

If you plan to build a custom UBI-based image rather than using the upstream pre-built image, you will also need [Git](https://git-scm.com/) and either [Podman](https://podman.io/) or access to OpenShift Builds.

## Planning decisions

A few decisions are easier to make before creating any resources, because changing them afterward requires redeployment or data migration.

### Choosing a storage class

OpenClaw writes JSONL session files and maintains SQLite databases with vector indexes. This I/O pattern requires POSIX file locking via `fcntl()`.

**Use block storage.** Your platform's default CSI-backed storage class is a good starting point. Block storage volumes are formatted with ext4 or xfs and provide correct locking semantics.

**Avoid network file systems.** NFS-backed storage classes (such as `nfs-client`, Azure Files, or Manila CSI shares) do not reliably support the `fcntl()` locking that SQLite requires. Using NFS can result in `SQLITE_BUSY` errors, `SQLITE_IOERR` failures, or silent data corruption. For more details, see the [Red Hat Knowledgebase article on SQLite and NFS](https://access.redhat.com/solutions/120733).

For sizing, 1Gi is a reasonable starting point. Session transcripts and the memory index grow over time, especially if you enable embeddings.

For production deployments, consider creating a storage class with `reclaimPolicy: Retain` so that persistent volume data survives accidental PVC deletion.

### Selecting an authentication mode

The gateway must authenticate incoming WebSocket connections when bound to all interfaces. OpenClaw supports three authentication modes:

| Mode | How it works | When to use |
|------|-------------|-------------|
| `token` | Shared secret stored in a Kubernetes Secret. Clients pass it in the WebSocket handshake. | Simplest option. Suitable for single-user or CI/CD workloads. |
| `password` | Shared password, similar to token mode. | Small teams with shared access. |
| `trusted-proxy` | A reverse proxy authenticates users and sets an `X-Forwarded-User` header. The gateway trusts the header. | Multi-user environments with OpenShift SSO integration. |

This post uses `token` mode. Integrating with OpenShift's built-in OAuth server using the oauth-proxy sidecar pattern is covered in a [later section](#integrating-with-openshift-sso-using-oauth-proxy).

### Understanding channel connectivity

Most messaging channels initiate outbound connections from the pod. This determines whether the Route needs to be publicly accessible:

| Channel | Connection type | Route needs public DNS? |
|---------|----------------|------------------------|
| Telegram (polling, default) | Outbound long-poll to Bot API | No |
| Discord | Outbound WebSocket to Discord gateway | No |
| Signal | Outbound SSE to signal-cli REST API | No |
| WhatsApp | Outbound WebSocket via Baileys | No |
| Telegram (webhook mode) | Inbound HTTP POST from Telegram | Yes |
| Slack (HTTP mode) | Inbound HTTP POST from Slack | Yes |

If you plan to use a polling-based channel (such as Telegram in its default mode), no additional network configuration is needed. If your deployment requires webhook-based channels, ensure that the Route has a public DNS entry and that NetworkPolicy rules permit inbound traffic from the channel provider.

## Getting a container image

You have two options: use the upstream pre-built image from GitHub Container Registry, or build an OpenShift-optimized image with Red Hat Universal Base Image (UBI). We cover both.

### Option A: Use the upstream image (no build required)

The OpenClaw project publishes multi-architecture images (AMD64 and ARM64) to GitHub Container Registry on every release:

```
ghcr.io/openclaw/openclaw:latest    # Latest stable release
ghcr.io/openclaw/openclaw:main      # Latest build from the main branch
ghcr.io/openclaw/openclaw:2026.2.26 # Specific version
```

These images are publicly available with no authentication required. They are based on the community `node:22-bookworm` image and run as the `node` user (UID 1000, GID 1000).

**Running the upstream image on OpenShift** requires one accommodation. OpenShift's `restricted-v2` SCC assigns an arbitrary UID at runtime (from the namespace's allocated range), not UID 1000. The arbitrary UID is always a member of GID 0 (the root group), but the upstream image's files are owned by GID 1000. The application files in `/app` are world-readable, so the gateway process can read them without issue. The writable state directory is the only concern, and we address it by pointing `OPENCLAW_STATE_DIR` to the PVC mount, where OpenShift's `fsGroup` mechanism ensures the correct group ownership.

In the Deployment spec (shown in a [later section](#step-5-create-the-deployment)), this means setting two environment variables:

```yaml
env:
  - name: OPENCLAW_STATE_DIR
    value: /data/openclaw
  - name: HOME
    value: /tmp
```

Setting `HOME` to `/tmp` prevents the gateway from attempting to write to `/home/node`, which is not writable under the arbitrary UID. The `/tmp` directory is always writable in a pod.

**Tradeoffs of the upstream image:**
- No build step required. Fastest way to get started.
- Larger image (~1.1 GB uncompressed, based on Debian Bookworm).
- Base OS packages are not covered by Red Hat security errata.

For production deployments where image provenance and Red Hat CVE coverage matter, consider building a UBI-based image instead.

### Option B: Build an OpenShift-optimized image with UBI

For production, we recommend building a custom image based on UBI. UBI provides several advantages in this context:

- **Designed for OpenShift's security model.** The UBI Node.js images run as UID 1001 with GID 0 (the root group), and set `g=u` permissions on writable directories. This aligns directly with how the `restricted-v2` SCC works. No additional Dockerfile modifications are needed to support arbitrary user IDs.
- **Covered by Red Hat security errata.** CVE patches for base OS packages follow Red Hat's security response process.
- **Smaller runtime image.** The minimal variant (`ubi9/nodejs-22-minimal`) produces a runtime image of approximately 200 MB, compared to roughly 1.1 GB for the upstream Debian-based image.
- **Freely redistributable.** UBI images on `registry.access.redhat.com` require no authentication and can be redistributed without a subscription.

We use a multi-stage build. The full image (`ubi9/nodejs-22`) serves as the build stage because it includes gcc, make, and git, which are needed to compile native Node.js addons such as `sharp` and `sqlite-vec`. The minimal image (`ubi9/nodejs-22-minimal`) serves as the runtime stage.

> **Note:** Red Hat's Node.js RPM packages do not include corepack. We install pnpm explicitly using `npm install -g pnpm` in the build stage.

Start by cloning the OpenClaw source repository:

```bash
git clone https://github.com/openclaw/openclaw.git
cd openclaw
```

Create a file called `Dockerfile.openshift` in the root of the cloned repository:

```dockerfile
# ── Stage 1: Build ──────────────────────────────────────────────
FROM registry.access.redhat.com/ubi9/nodejs-22 AS build

WORKDIR /opt/app-root/src

# Copy package.json first to read the packageManager field,
# then install the exact pnpm version it declares.
COPY --chown=1001:0 package.json ./
USER 0
RUN PNPM_VERSION=$(node -p "require('./package.json').packageManager?.split('@')[1] || '10'") && \
    npm install -g "pnpm@$PNPM_VERSION" && \
    chown -R 1001:0 /opt/app-root/src/.npm
USER 1001

# All COPY instructions use --chown=1001:0 so that pnpm
# (running as UID 1001) can create node_modules subdirectories.
COPY --chown=1001:0 pnpm-lock.yaml pnpm-workspace.yaml .npmrc ./
COPY --chown=1001:0 ui/package.json ./ui/package.json
COPY --chown=1001:0 patches ./patches
COPY --chown=1001:0 scripts ./scripts

# Install without postinstall scripts, then selectively rebuild
# only the native addons the gateway needs. node-llama-cpp is
# skipped: it requires cmake and llama.cpp compilation for local
# LLM inference, which is not needed when connecting to remote
# model providers.
RUN NODE_OPTIONS=--max-old-space-size=2048 pnpm install --frozen-lockfile --ignore-scripts && \
    pnpm rebuild esbuild sharp koffi protobufjs

COPY --chown=1001:0 . .
# Build the A2UI canvas bundle. If this fails (e.g. cross-platform
# QEMU builds), create a stub and remove sources so the build
# script's fallback path succeeds. The gateway works without it.
RUN pnpm canvas:a2ui:bundle || \
    (echo "A2UI bundle: creating stub (non-fatal)" && \
     mkdir -p src/canvas-host/a2ui && \
     echo "/* A2UI bundle unavailable */" > src/canvas-host/a2ui/a2ui.bundle.js && \
     echo "stub" > src/canvas-host/a2ui/.bundle.hash && \
     rm -rf vendor/a2ui apps/shared/OpenClawKit/Tools/CanvasA2UI)
RUN pnpm build

ENV OPENCLAW_PREFER_PNPM=1
RUN pnpm ui:build

# ── Stage 2: Runtime ───────────────────────────────────────────
FROM registry.access.redhat.com/ubi9/nodejs-22-minimal

LABEL org.opencontainers.image.source="https://github.com/openclaw/openclaw" \
      org.opencontainers.image.title="OpenClaw (OpenShift)" \
      org.opencontainers.image.description="OpenClaw gateway on UBI 9 Node.js 22 minimal"

WORKDIR /opt/app-root/src

COPY --from=build /opt/app-root/src/dist ./dist
COPY --from=build /opt/app-root/src/node_modules ./node_modules
COPY --from=build /opt/app-root/src/package.json .
COPY --from=build /opt/app-root/src/openclaw.mjs .
COPY --from=build /opt/app-root/src/extensions ./extensions

USER 0
RUN mkdir -p /data/openclaw && \
    chgrp -R 0 /data/openclaw && \
    chmod -R g=u /data/openclaw && \
    ln -sf /opt/app-root/src/openclaw.mjs /usr/local/bin/openclaw && \
    chmod 755 /opt/app-root/src/openclaw.mjs
USER 1001

ENV NODE_ENV=production
ENV OPENCLAW_STATE_DIR=/data/openclaw

EXPOSE 18789

HEALTHCHECK --interval=3m --timeout=10s --start-period=15s --retries=3 \
  CMD node -e "fetch('http://127.0.0.1:18789/healthz').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"

CMD ["node", "openclaw.mjs", "gateway", "--allow-unconfigured"]
```

Build the image from the repository root. You can build locally with Podman and push to a registry, or build directly on the cluster using an OpenShift BuildConfig:

```bash
# Option A: Build locally with Podman and push to a registry.
# The --memory flag prevents OOM kills during dependency installation.
podman build --memory=4g -f Dockerfile.openshift -t quay.io/yourorg/openclaw:latest .
podman push quay.io/yourorg/openclaw:latest

# Option B: Build on the cluster using OpenShift Builds
oc new-project openclaw
oc new-build --name openclaw --dockerfile - < Dockerfile.openshift
oc start-build openclaw --from-dir=. --follow
```

Both options run from the root of the cloned `openclaw` repository, since the `COPY` instructions in the Dockerfile reference files relative to that directory.

## Deploying step by step

This section creates each Kubernetes resource individually so that you can see exactly what goes onto the cluster. If you prefer a single-command deployment, skip ahead to the [Helm chart](#deploying-with-a-helm-chart) section.

The YAML manifests below are self-contained. You can save each one to a file and apply it with `oc apply -f`, or apply them all from a single directory. No files from the OpenClaw repository are needed for this section -- the container image built in the previous step contains everything the gateway needs to run.

### Step 1: Create the project

```bash
oc new-project openclaw
```

### Step 2: Create the Secret

Store the gateway authentication token and model provider API keys in a Kubernetes Secret. Add channel credentials as needed:

```bash
GATEWAY_TOKEN=$(openssl rand -hex 32)

oc create secret generic openclaw-secrets \
  --from-literal=OPENCLAW_GATEWAY_TOKEN="$GATEWAY_TOKEN" \
  --from-literal=ANTHROPIC_API_KEY="your-anthropic-api-key"
  # Optional: add channel credentials
  # --from-literal=TELEGRAM_BOT_TOKEN="your-telegram-bot-token"

echo "Save this gateway token for CLI configuration: $GATEWAY_TOKEN"
```

### Step 3: Create the ConfigMap

The gateway reads its configuration from a JSON5 file. Store it in a ConfigMap:

```yaml
# openclaw-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: openclaw-config
  namespace: openclaw
data:
  openclaw.json: |
    {
      // Bind to all interfaces. Required inside a pod,
      // since loopback (the default) is not reachable
      // from the Service or Route.
      "gateway": {
        "bind": "lan",
        "port": 18789,
        "auth": {
          "mode": "token"
          // Token value is read from OPENCLAW_GATEWAY_TOKEN env var.
        }
      },
      // Optional: enable one or more channels.
      // Telegram example (requires TELEGRAM_BOT_TOKEN in the Secret):
      // "channels": {
      //   "telegram": {
      //     "enabled": true,
      //     "accounts": {
      //       "default": {}
      //     }
      //   }
      // }
    }
```

```bash
oc apply -f openclaw-config.yaml
```

### Step 4: Create the PersistentVolumeClaim

```yaml
# openclaw-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: openclaw-data
  namespace: openclaw
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  # Omitting storageClassName uses the cluster default.
```

```bash
oc apply -f openclaw-pvc.yaml
```

### Step 5: Create the Deployment

```yaml
# openclaw-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: openclaw-gateway
  namespace: openclaw
  labels:
    app: openclaw
    component: gateway
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: openclaw
      component: gateway
  template:
    metadata:
      labels:
        app: openclaw
        component: gateway
    spec:
      securityContext:
        runAsNonRoot: true
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: gateway
          # If using the upstream image (Option A):
          #   image: ghcr.io/openclaw/openclaw:latest
          # If using the UBI-based image (Option B):
          image: quay.io/yourorg/openclaw:latest
          ports:
            - containerPort: 18789
              name: gateway
              protocol: TCP
          env:
            - name: OPENCLAW_STATE_DIR
              value: /data/openclaw
            - name: OPENCLAW_CONFIG_PATH
              value: /etc/openclaw/openclaw.json
            # Required when using the upstream image, to prevent
            # writes to /home/node which is not writable under
            # OpenShift's arbitrary UID assignment.
            # Harmless when using the UBI-based image.
            - name: HOME
              value: /tmp
            - name: NODE_OPTIONS
              value: "--max-old-space-size=768"
          envFrom:
            - secretRef:
                name: openclaw-secrets
          volumeMounts:
            - name: data
              mountPath: /data/openclaw
            - name: config
              mountPath: /etc/openclaw
              readOnly: true
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop:
                - ALL
          livenessProbe:
            httpGet:
              path: /healthz
              port: 18789
            initialDelaySeconds: 15
            periodSeconds: 30
            timeoutSeconds: 5
          readinessProbe:
            httpGet:
              path: /readyz
              port: 18789
            initialDelaySeconds: 5
            periodSeconds: 10
            timeoutSeconds: 5
          resources:
            requests:
              memory: 256Mi
              cpu: 250m
            limits:
              memory: 1Gi
              cpu: "1"
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: openclaw-data
        - name: config
          configMap:
            name: openclaw-config
```

```bash
oc apply -f openclaw-deployment.yaml
```

A few things to note about this Deployment:

**Recreate strategy.** This is required when using a `ReadWriteOnce` PVC with a single-replica Deployment. With the default `RollingUpdate` strategy, OpenShift creates the new pod before terminating the old one. The new pod attempts to mount the PVC, but the old pod still holds it, resulting in the new pod stuck in `ContainerCreating`. Meanwhile, OpenShift does not terminate the old pod because the new one is not yet Ready. The result is a permanent deadlock. The `Recreate` strategy avoids this by terminating the old pod first. The tradeoff is a brief window of downtime during upgrades. For more background, see the [Red Hat Knowledgebase article on rollout behavior with RWO volumes](https://access.redhat.com/solutions/7041563).

**Security context.** The pod-level `securityContext` sets `runAsNonRoot: true` and applies the `RuntimeDefault` seccomp profile. The container-level context disables privilege escalation and drops all Linux capabilities. We intentionally omit `runAsUser` so that OpenShift assigns a UID from the namespace's allocated range, which is how the `restricted-v2` SCC is designed to work. Because we built the image on UBI with GID 0 permissions, the assigned UID can read and write all necessary directories.

**No `runAsUser` field.** OpenShift's `restricted-v2` SCC uses the `MustRunAsRange` strategy for user IDs. Each namespace receives a unique, non-overlapping UID range through the `openshift.io/sa.scc.uid-range` annotation. If the pod spec omits `runAsUser`, OpenShift assigns the first UID in that range. If you explicitly set `runAsUser: 1001` (the UBI default), admission will reject the pod because 1001 falls outside the namespace's range. For more details, see the [Red Hat blog on OpenShift and UIDs](https://www.redhat.com/en/blog/a-guide-to-openshift-and-uids).

**ConfigMap mounted read-only.** The configuration file is mounted at `/etc/openclaw/openclaw.json` as a read-only ConfigMap volume, separate from the writable state directory at `/data/openclaw`. The `OPENCLAW_CONFIG_PATH` environment variable tells the gateway where to find it. This separation is important because the gateway performs atomic rename-writes when persisting configuration changes. If the config file were mounted inside the state directory using `subPath`, the rename operation would fail with `EBUSY`. Mounting the ConfigMap to its own read-only path avoids this entirely. The gateway logs a warning that it cannot persist runtime configuration changes, but this is expected and harmless — the ConfigMap remains the source of truth and any runtime values are held in memory.

**Node.js heap limit.** The `NODE_OPTIONS` environment variable sets the V8 heap limit to 768 MB, which fits within the 1 Gi container memory limit while leaving room for the Node.js runtime, native addons, and OS overhead. Without this setting, the default heap limit on some platforms may be too small for the gateway's startup initialization, resulting in an out-of-memory crash.

### Step 6: Create the Service

```yaml
# openclaw-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: openclaw-gateway
  namespace: openclaw
  labels:
    app: openclaw
    component: gateway
spec:
  type: ClusterIP
  ports:
    - name: gateway
      port: 18789
      targetPort: gateway
      protocol: TCP
  selector:
    app: openclaw
    component: gateway
```

```bash
oc apply -f openclaw-service.yaml
```

### Step 7: Create the Route

```yaml
# openclaw-route.yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: openclaw-gateway
  namespace: openclaw
  labels:
    app: openclaw
  annotations:
    haproxy.router.openshift.io/timeout: "3600s"
spec:
  to:
    kind: Service
    name: openclaw-gateway
    weight: 100
  port:
    targetPort: gateway
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
  wildcardPolicy: None
```

```bash
oc apply -f openclaw-route.yaml
```

OpenShift's HAProxy-based Ingress Controller handles WebSocket natively. After the HTTP upgrade handshake completes, HAProxy treats the connection as a bidirectional TCP tunnel. No special annotation is needed to enable WebSocket support. The `haproxy.router.openshift.io/timeout` annotation sets the idle timeout for these long-lived connections. The default tunnel timeout is one hour; we match that here.

We use edge TLS termination, which terminates TLS at the HAProxy router and proxies plain HTTP to the pod. This avoids HTTP/2 ALPN negotiation on the backend connection, which can interfere with the HTTP/1.1 WebSocket upgrade mechanism. Edge termination is the simplest and most reliable option for WebSocket workloads.

There is no HAProxy-level payload limit for WebSocket frames. After the upgrade handshake, data flows as a TCP stream without per-message buffering or size inspection. OpenClaw's 25 MB maximum WebSocket payload passes through without issue.

### Step 8: Apply network policies

By default, OpenShift allows all pod-to-pod and pod-to-external traffic. To restrict this, apply a deny-by-default policy with explicit allow rules.

```yaml
# openclaw-networkpolicy.yaml

# Deny all ingress and egress by default
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-by-default
  namespace: openclaw
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  ingress: []
  egress: []
---
# Allow ingress from the OpenShift Ingress Controller
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-openshift-ingress
  namespace: openclaw
spec:
  podSelector:
    matchLabels:
      app: openclaw
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              network.openshift.io/policy-group: ingress
---
# Allow DNS resolution
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: openclaw
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: openshift-dns
      ports:
        - protocol: UDP
          port: 5353
        - protocol: TCP
          port: 5353
---
# Allow outbound HTTPS to external APIs
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-https
  namespace: openclaw
spec:
  podSelector:
    matchLabels:
      app: openclaw
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 10.0.0.0/8
              - 172.16.0.0/12
              - 192.168.0.0/16
      ports:
        - protocol: TCP
          port: 443
```

```bash
oc apply -f openclaw-networkpolicy.yaml
```

**A note on DNS port numbers.** In OpenShift, CoreDNS pods listen on container port 5353. The `dns-default` Service in the `openshift-dns` namespace maps Service port 53 to target port 5353. Because NetworkPolicy egress rules match against the destination pod's container port rather than the Service port, the DNS egress rule above correctly targets port 5353. Using port 53 in a NetworkPolicy egress rule would silently prevent DNS resolution.

**Tighter egress control with EgressFirewall.** The policy above allows outbound HTTPS to any external IP. For more granular control, OpenShift provides the `EgressFirewall` custom resource through OVN-Kubernetes. EgressFirewall supports DNS-based rules, enabling you to allow traffic only to specific external services:

```yaml
# openclaw-egressfirewall.yaml (optional, requires cluster-admin)
apiVersion: k8s.ovn.org/v1
kind: EgressFirewall
metadata:
  name: default
  namespace: openclaw
spec:
  egress:
    - type: Allow
      to:
        dnsName: api.telegram.org
    - type: Allow
      to:
        dnsName: gateway.discord.gg
    - type: Allow
      to:
        dnsName: api.anthropic.com
    - type: Allow
      to:
        dnsName: api.openai.com
    - type: Deny
      to:
        cidrSelector: 0.0.0.0/0
```

Note that each namespace supports a single EgressFirewall resource, which must be named `default`. Creating an EgressFirewall requires cluster-admin privileges. For additional details, see [Configuring an egress firewall for a project](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/network_security/egress-firewall) in the OpenShift documentation.

## Deploying with a Helm chart

If you prefer a single-command deployment, the manifests from the previous section can be packaged as a Helm chart. The OpenClaw project provides an example chart in its repository under `deploy/openshift/chart/`. If you prefer to create your own, the key `values.yaml` structure maps directly to the resources we created manually:

```yaml
image:
  repository: quay.io/yourorg/openclaw
  tag: latest

gateway:
  port: 18789
  bind: lan
  auth:
    mode: token

secrets:
  gatewayToken: ""       # Set via --set or external secrets operator
  anthropicApiKey: ""
  # telegramBotToken: ""  # Optional: add channel credentials

storage:
  size: 1Gi
  # storageClassName: gp3-csi  # Uncomment to override the default

route:
  enabled: true
  tls:
    termination: edge
  timeout: "3600s"

resources:
  requests:
    memory: 256Mi
    cpu: 250m
  limits:
    memory: 1Gi
    cpu: "1"

networkPolicy:
  enabled: true
```

Install with:

```bash
helm install openclaw ./chart \
  -n openclaw --create-namespace \
  --set secrets.gatewayToken="$(openssl rand -hex 32)" \
  --set secrets.anthropicApiKey="your-key"
```

The chart creates the same set of resources as the manual path: Secret, ConfigMap, PVC, Deployment with `Recreate` strategy, Service, Route, and NetworkPolicy rules.

## Verifying the deployment

With all resources applied, verify that the gateway is running and reachable.

### Check the pod

```bash
oc get pods -n openclaw
```

Expected output:

```
NAME                                READY   STATUS    RESTARTS   AGE
openclaw-gateway-5d4f8b7c9-x2k4m   1/1     Running   0          2m
```

### Inspect the logs

```bash
oc logs deploy/openclaw-gateway -n openclaw
```

Look for the startup message confirming that the gateway is listening:

```
Gateway listening on 0.0.0.0:18789
```

If you configured a channel (for example, Telegram), you should also see it connect:

```
Telegram: connected (polling)
```

### Test the health endpoints

Retrieve the Route hostname and test the liveness and readiness probes:

```bash
ROUTE_HOST=$(oc get route openclaw-gateway -n openclaw -o jsonpath='{.spec.host}')

curl -s "https://$ROUTE_HOST/healthz" | jq .
# {"ok":true,"status":"live"}

curl -s "https://$ROUTE_HOST/readyz" | jq .
# {"ok":true,"status":"ready"}
```

### Connect the OpenClaw CLI

To interact with the gateway from your workstation, install the OpenClaw CLI and use port-forwarding:

```bash
# Install the CLI (requires Node.js 22+)
npm install -g openclaw

# Forward the gateway port to your local machine
oc port-forward svc/openclaw-gateway 18789:18789 -n openclaw
```

In another terminal, configure the CLI to connect to the forwarded port:

```bash
openclaw config set gateway.remote.url ws://localhost:18789
openclaw config set gateway.remote.token "$GATEWAY_TOKEN"
openclaw channels status --probe
```

For other installation methods, see the [OpenClaw installation documentation](https://docs.openclaw.ai/install).

### Send a test message (if a channel is configured)

If you configured a messaging channel, send a test message through it. For example, send a message to your Telegram bot. The message should appear in the gateway logs and you should receive a response. If this is the first message from your account, OpenClaw will respond with a pairing code, as the DM policy defaults to `pairing` mode.

If no channel is configured yet, you can still interact with the gateway through the CLI using `openclaw message send`.

## Day-2 operations

### Upgrading

Build a new image from the updated source (pull the latest changes, rebuild with Podman or OpenShift Builds as described earlier), then update the Deployment to reference the new tag:

```bash
oc set image deploy/openclaw-gateway \
  gateway=quay.io/yourorg/openclaw:v2026.3.1 \
  -n openclaw
```

The `Recreate` strategy terminates the old pod before starting the new one, resulting in a brief window of downtime. The gateway handles `SIGTERM` gracefully: it stops accepting new work, waits up to 30 seconds for in-flight agent turns to complete, then shuts down.

### Backing up state

All persistent state lives on the PVC. Create a volume snapshot:

```bash
oc apply -f - <<'EOF'
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: openclaw-backup
  namespace: openclaw
spec:
  source:
    persistentVolumeClaimName: openclaw-data
EOF
```

### Monitoring resource consumption

The gateway process is CPU-light, but memory usage grows with the number of active agents and the size of session transcripts held in memory. The session store maintains a 45-second in-memory cache, and the memory index manager keeps open SQLite handles per agent.

```bash
oc adm top pods -n openclaw
```

If you observe memory pressure, increase the memory limit in the Deployment's resource specification.

### Understanding restart behavior

The gateway acquires a lock file in `/tmp` to prevent multiple instances from binding to the same port. In a pod, `/tmp` is ephemeral storage that is cleared on restart. No manual cleanup is needed after a pod restart or reschedule.

## Integrating with OpenShift SSO using oauth-proxy

For environments where multiple users need access to the gateway's control UI, you can integrate with OpenShift's built-in OAuth server using the [oauth-proxy](https://github.com/openshift/oauth-proxy) sidecar pattern. The proxy authenticates users against the OpenShift API and passes the authenticated username to the gateway through the `X-Forwarded-User` header.

> **WebSocket consideration.** The oauth-proxy detects WebSocket upgrade requests using a case-sensitive comparison on the `Connection` header value. Browser-based WebSocket clients send `Connection: Upgrade` (capitalized) and work as expected. Some server-to-server WebSocket clients may send a lowercase value, which the proxy does not recognize as a WebSocket upgrade. If you encounter this, see [oauth-proxy issue #163](https://github.com/openshift/oauth-proxy/issues/163) for details and workarounds.

### Required resources

**ServiceAccount** with an OAuth redirect annotation:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: openclaw-proxy
  namespace: openclaw
  annotations:
    serviceaccounts.openshift.io/oauth-redirectreference.primary: >-
      {"kind":"OAuthRedirectReference","apiVersion":"v1",
       "reference":{"kind":"Route","name":"openclaw-gateway"}}
```

**ClusterRoleBinding** for authentication delegation:

```bash
oc create clusterrolebinding openclaw-auth-delegator \
  --clusterrole=system:auth-delegator \
  --serviceaccount=openclaw:openclaw-proxy
```

**Updated Service** with a serving certificate annotation. OpenShift's service-ca controller automatically generates a TLS certificate and stores it in the named Secret:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: openclaw-gateway
  namespace: openclaw
  annotations:
    service.alpha.openshift.io/serving-cert-secret-name: openclaw-proxy-tls
spec:
  ports:
    - name: proxy
      port: 443
      targetPort: 8443
  selector:
    app: openclaw
    component: gateway
```

**Updated Route** with re-encrypt TLS termination, so that traffic is encrypted between the router and the oauth-proxy sidecar:

```yaml
spec:
  tls:
    termination: Reencrypt
```

### Deployment changes

Add the oauth-proxy sidecar container to the pod specification:

```yaml
containers:
  - name: oauth-proxy
    image: registry.redhat.io/openshift4/ose-oauth-proxy:latest
    ports:
      - containerPort: 8443
        name: public
    args:
      - --https-address=:8443
      - --provider=openshift
      - --openshift-service-account=openclaw-proxy
      - --upstream=http://localhost:18789
      - --tls-cert=/etc/tls/private/tls.crt
      - --tls-key=/etc/tls/private/tls.key
      - --cookie-secret=<generate-a-32-byte-base64-string>
      - --skip-auth-regex=^/healthz$
      - --skip-auth-regex=^/readyz$
    volumeMounts:
      - name: proxy-tls
        mountPath: /etc/tls/private
  - name: gateway
    # ... same as before
volumes:
  - name: proxy-tls
    secret:
      secretName: openclaw-proxy-tls
```

Update the gateway configuration to trust the proxy:

```json5
{
  "gateway": {
    "auth": {
      "mode": "trusted-proxy",
      "trustedProxy": {
        "userHeader": "X-Forwarded-User"
      }
    }
  }
}
```

> **Important:** Use the `ose-oauth-proxy` image from `registry.redhat.io`. The `openshift/oauth-proxy` image on Docker Hub has not been updated for recent OpenShift releases and will fail on current clusters due to removed Kubernetes API versions.

## Key considerations

Here is a summary of the platform-specific considerations covered in this post:

- **Storage selection matters.** SQLite requires POSIX `fcntl()` locking. Block storage provides this correctly. Network file systems such as NFS do not. See the [Red Hat Knowledgebase](https://access.redhat.com/solutions/120733) for details.
- **Use the `Recreate` deployment strategy** when a Deployment uses a `ReadWriteOnce` PVC. The `RollingUpdate` strategy creates a scheduling deadlock with single-replica stateful workloads. See [Red Hat Solution 7041563](https://access.redhat.com/solutions/7041563).
- **NetworkPolicy DNS rules target container ports.** On OpenShift, CoreDNS listens on container port 5353, not 53. NetworkPolicy egress rules that reference port 53 will not match CoreDNS pod traffic.
- **UBI images handle arbitrary UIDs.** When building container images for OpenShift, use UBI as the base and ensure writable directories are owned by GID 0 with group-write permissions. Do not set `runAsUser` in the pod spec.
- **Mount ConfigMap volumes read-only at a separate path.** The gateway performs atomic rename-writes to its configuration file. Mounting a ConfigMap using `subPath` inside the state directory causes `EBUSY` errors. Use a dedicated read-only mount and set `OPENCLAW_CONFIG_PATH` to point to it.
- **Set the Node.js heap limit explicitly.** The default V8 heap limit may be too small on some platforms. Set `NODE_OPTIONS=--max-old-space-size=768` (or higher) to prevent out-of-memory crashes during startup.
- **The oauth-proxy image source matters.** Always use `registry.redhat.io/openshift4/ose-oauth-proxy` for current OpenShift releases.

## Additional resources

- [OpenClaw documentation](https://docs.openclaw.ai)
- [OpenClaw on GitHub](https://github.com/openclaw/openclaw)
- [Red Hat Universal Base Images (UBI)](https://www.redhat.com/en/blog/introducing-red-hat-universal-base-image)
