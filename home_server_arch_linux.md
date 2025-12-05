First, install Arch Linux as mentioned in arch_install.md
Some differences:

1. no need for desktop environment.
2. mask the bluetooth service to reduce security risks from bluetooth.
3. use UTC timezone & en_US.UTF-8 locale.

# Docker & Deployment Architecture

**Scenario:**
- **Server:** Dell G15 (Old Laptop) hosting website, apps, bots.
- **Dev Machine:** New Laptop.
- **Exposure:** Cloudflare Tunnel.
- **Constraint:** Free Docker Hub account (1 private image limit).
- **Goal:** Avoid building images on the old server; keep code on the new laptop.

## Core Components
Regardless of the deployment strategy, these components are essential:
1.  **Local Docker Registry:** Run a registry container on the Dell G15 to bypass Docker Hub limits.
    - *Workflow:* Build on New Laptop -> Push to Dell G15 Registry IP -> Dell G15 pulls from `localhost`.
2.  **Cloudflare Tunnel:** Exposes services securely.
3.  **Shared Network:** A Docker network (e.g., `tunnellink`) to connect apps to the Cloudflare tunnel.

---

## Strategy Options

### Option 1: Remote Control (Docker Context) - *Recommended for Workflow*
Keep `docker-compose.yml` files on the **New Laptop**. Treat the Dell G15 purely as a runtime target.

**Setup:**
1.  **Context:** On New Laptop, run `docker context create home-server --docker "host=ssh://user@<server-ip>"` then `docker context use home-server`.
2.  **Build & Push:**
    ```bash
    docker build -t <server-ip>:5000/my-app .
    docker push <server-ip>:5000/my-app
    ```
3.  **Deploy:** Run `docker compose up -d` on New Laptop. The containers spin up on the Dell G15.

**Pros:**
*   **Speed:** Single step deployment.
*   **Git Integration:** Compose files stay in your repo on the dev machine.
*   **Clean Server:** No config files or source code cluttering the server.

**Cons:**
*   **Dependency:** Need the laptop to change config/restart specific services comfortably.

---

### Option 2: Server-Side Monolith
Maintain a single `docker-compose.yml` on the **Dell G15** containing *everything* (Registry, Tunnel, Apps).

**Workflow:**
1.  Build/Push on New Laptop.
2.  SSH into Dell G15.
3.  Edit `docker-compose.yml` (if needed).
4.  Run `docker compose pull && docker compose up -d`.

**Pros:**
*   **Independence:** Server is self-contained; easy to restart/recover without the laptop.
*   **Simplicity:** One file to rule them all.

**Cons:**
*   **Risk:** Restarting the compose stack reloads *everything* (Tunnel/Registry), potentially disrupting access while updating a minor app.
*   **Friction:** Requires SSH + Manual Edit + Pull steps for every update.

---

### Option 3: Server-Side Hybrid - *Recommended for Stability*
Similar to Option 2, but split into two separate compose files on the **Dell G15**:
1.  `infra/docker-compose.yml`: **Cloudflared** & **Local Registry**. (Rarely changes).
2.  `apps/docker-compose.yml`: **Custom Apps** & **Bots**. (Frequent updates).

**Workflow:**
1.  Build/Push on New Laptop.
2.  SSH into Dell G15.
3.  Run `docker compose -f apps/docker-compose.yml pull && docker compose -f apps/docker-compose.yml up -d`.

#### Managing Individual Applications
When using a separate `apps/docker-compose.yml` file, you can manage individual services within it without affecting other apps or the infrastructure.

*   **Restart a specific app:**
    ```bash
    cd /path/to/apps # Navigate to where apps/docker-compose.yml is located
    docker compose -f docker-compose.yml restart <service_name>
    ```
    (Replace `<service_name>` with the name of the service in your compose file, e.g., `my-web-app`).

*   **Stop a specific app:**
    ```bash
    cd /path/to/apps
    docker compose -f docker-compose.yml stop <service_name>
    ```

*   **Start a specific app:**
    ```bash
    cd /path/to/apps
    docker compose -f docker-compose.yml start <service_name>
    ```

This granular control ensures that updating or troubleshooting one application does not inadvertently disrupt other running services. Moreover, Watchtower (if configured) will automatically perform these restarts for updated images, eliminating the need for manual intervention.

**Pros:**
*   **Stability:** Updating an app will never accidentally kill the Tunnel or Registry.
*   **Organization:** Clear separation of concerns.

**Cons:**
*   **Management:** Two files to manage instead of one.

---

## Comparison Summary

| Feature | Option 1 (Context) | Option 2 (Monolith) | Option 3 (Hybrid) |
| :--- | :--- | :--- | :--- |
| **Deployment Speed** | **Fastest** (Remote command) | **Slow** (SSH + Pull) | **Slow** (SSH + Pull) |
| **Server Config** | Zero config on server | Single file on server | Multiple files on server |
| **Reliability** | High (Laptop dependent for updates) | Medium (Risk of full stack restart) | **Highest** (Isolated failures) |
| **Best For** | Rapid Iteration / Dev | Simple / Set-and-Forget | Production-like Stability |

---

## Automating Updates (Continuous Deployment)

Manually SSHing to the server to run `docker compose pull` (Options 2 & 3) is tedious. These tools automate that process.

### 1. Watchtower (Polling) - *Recommended*
A container that runs on your server, periodically checks your registry for updated images, and automatically restarts containers with the new image.

*   **Pros:** Extremely simple (one container); no complex network/webhook setup.
*   **Cons:** Updates are not instant (delayed by polling interval).
*   **Setup:** Add to your `infra/docker-compose.yml`:
    ```yaml
    watchtower:
      image: containrrr/watchtower
      volumes:
        - /var/run/docker.sock:/var/run/docker.sock
        # - /root/.docker/config.json:/config.json # (Optional) If using auth for registry
      command: --interval 300 # Check every 5 minutes
      restart: unless-stopped
    ```

### 2. Webhooks (Push-Based) - *Advanced*
Trigger an update script on the server immediately after a build.

*   **Workflow:** Build finishes on New Laptop -> Script curls `http://<server-ip>:9000/hooks/redeploy` -> Server runs `docker compose pull && up`.
*   **Pros:** Instant updates.
*   **Cons:** Requires running a webhook listener (e.g., `adnanh/webhook`) and managing security/firewalls for that listener.

---

## Managing Secrets

Handling API keys, passwords, and tokens securely.

### 1. Environment Files (`.env`) - *Simplest*
Store secrets in a `.env` file next to your `docker-compose.yml`.
*   **Pros:** Very easy to use; Docker Compose picks it up automatically.
*   **Cons:** The file sits in plain text on the disk. If you check `docker inspect`, env vars are visible.
*   **Best for:** Home labs where physical access is controlled.

### 2. Docker Secrets - *More Secure*
Docker's native way to handle sensitive data.
*   **Pros:** Secrets are not exposed in environment variables (safer from `docker inspect`). Files are mounted into `/run/secrets/`.
*   **Cons:** Slightly more verbose configuration in `docker-compose.yml`.
*   **Setup:**
    ```yaml
    services:
      myapp:
        secrets:
          - db_password
    secrets:
      db_password:
        file: ./secrets/db_password.txt
    ```

### 3. Secret Management Services (e.g., Bitwarden, HashiCorp Vault) - *Overkill*
Injecting secrets at runtime from an external vault.
*   **Pros:** Centralized, highly secure, audit logs.
*   **Cons:** Massive complexity overhead for a home server.

---

## Zero-Downtime Rolling Updates

Achieving Kubernetes-like rolling updates on a single Docker host without an orchestrator like Swarm or Kubernetes requires careful configuration. The key is to leverage Docker Compose's `deploy` and `healthcheck` configurations, especially when using an internal Docker network (like `tunnellink`) with Cloudflare Tunnel.

**The Strategy:**

1.  **Start New First:** Tell Docker Compose to start the new version of a container *before* stopping the old version.
2.  **Health Check:** Ensure the new version is healthy and ready to receive traffic before the old version is removed.
3.  **Internal Networking:** Rely on Docker's internal DNS and your Cloudflare Tunnel to route traffic, avoiding host port conflicts.

**How it works during `docker compose up -d`:**

1.  Docker Compose notices an image change for a service.
2.  It starts a new container (e.g., `my-app-v2`) with the updated image.
3.  The old container (`my-app-v1`) continues to run and serve traffic.
4.  Docker waits for `my-app-v2` to pass its configured `healthcheck`.
5.  Once `my-app-v2` is healthy, Docker Compose gradually stops and removes `my-app-v1`.
6.  Traffic (via Cloudflare Tunnel and the internal Docker network) is seamlessly shifted to `my-app-v2`.

**Example `docker-compose.yml` configuration (for an app service):**

```yaml
services:
  code-runner:
    image: localhost:5000/code-runner:latest # Your app image from local registry
    networks:
      - tunnellink # Connect to your shared tunnel network
    deploy:
      mode: replicated # Required for update_config options
      replicas: 1 # You can increase this for more instances if needed, but 1 is fine for rolling update
      update_config:
        order: start-first # Critical: Start new container BEFORE stopping old
        failure_action: rollback # What to do if the new container fails health checks
        delay: 10s # Delay between steps (starting new, stopping old)
      # restart_policy: # Optional: Define how instances restart on failure
      #   condition: on-failure
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"] # Your app's health endpoint
      interval: 10s # How often to check
      timeout: 5s # How long to wait for a response
      retries: 3 # How many times to retry before marking unhealthy
      start_period: 5s # Grace period for container to start before health checks begin
    # No 'ports' section for host-binding!
    # The Cloudflare Tunnel container will access this service via its internal Docker network name.

networks:
  tunnellink:
    external: true # Use the network defined in your infra-compose.yml
```

**Key Considerations:**

*   **No Host Port Binding:** For this to work on a single host, your application containers **must NOT bind ports directly to the host machine**. The Cloudflare Tunnel container (`cloudflared`) communicates with your applications over the internal Docker network (e.g., `tunnellink`) using their service names (e.g., `http://code-runner:8080`). This avoids port conflicts when both old and new versions are briefly running.
*   **Health Checks are Crucial:** A robust health check (`test` command) that accurately determines if your application is fully initialized and ready to serve traffic is vital for this strategy to work effectively.
*   **`deploy` Section:** The `deploy` section within a service definition enables these advanced features, typically used in Docker Swarm mode but also useful for single-host rolling updates in recent Docker Compose versions.
*   **Automation:** When combined with **Watchtower**, this becomes fully automated. Watchtower pulls the new image and then triggers a `docker compose up -d` internally, which will execute this rolling update strategy.