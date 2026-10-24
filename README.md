# Minecraft Router — Deablr

Routes Minecraft players to the Deablr Minecraft servers based on the hostname they connect with, so multiple servers can share a single public IP and the standard Minecraft port (`25565`).

Java Edition players connect to the router using a domain name, and `itzg/mc-router` forwards the connection to the correct backend server.

Bedrock Edition players connect to Geyser (`geyser/`, bundled in this project), which translates them into Java clients and hands them to `mc-router`, so Bedrock and Java players land on the same servers. See [Bedrock support](#bedrock-support).

| Domain | Backend Container | Description | Bedrock port |
|--------|-------------------|-------------|--------------|
| `richard.deablr.com` | `richard-deablr-mc:25565` | Richard's creative flat Paper server | `19132/udp` |
| `nijika.deablr.com` | `nijika-deablr-mc:25565` | Nijika's Paper server | `19132/udp` |

Java Edition players use port `25565` (the default).

## How It Works

1. `mc-router` listens on host port `25565`.
2. A Minecraft client connects to `richard.deablr.com:25565` or `nijika.deablr.com:25565`.
3. `mc-router` reads the Server Name Indication (SNI) hostname and proxies the connection to the matching backend container.
4. The backend containers are reached over the shared `dokploy-network` Docker network.

## Bedrock support

Bedrock Edition players join the same servers through a shared [Geyser](https://geysermc.org/) Standalone instance that runs in this compose project.

```
Bedrock client ──UDP 19132──▶ geyser-deablr
                                  │ joins mc-router as a Java client,
                                  │ forwarding the hostname the player typed
                                  ▼
                              mc-router ──▶ routes by hostname, same as Java
```

- Players add `nijika.deablr.com` (or `richard.deablr.com`) as a Bedrock server with the default port `19132`. No per-port or per-server Bedrock setup is needed; adding a new server only requires a new `MAPPING` entry.
- Geyser runs with `java.forward-hostname: true`, so the hostname the Bedrock player typed is forwarded to `mc-router`, which routes exactly as it does for Java.
- Bedrock players authenticate through [Floodgate](https://geysermc.org/wiki/floodgate/) (`java.auth-type: floodgate`) and appear on backends with a `.` prefix (e.g. `.Steve`).
- Geyser's `motd.passthrough-*` settings are off because a Bedrock server-list ping carries no hostname and `mc-router` could not route it.

### Shared Floodgate key

Geyser and every Bedrock-enabled backend must share the same Floodgate `key.pem`. It lives once on the VPS at `/opt/deablr/floodgate/key.pem` (16 random bytes, never committed) and is mounted read-only into Geyser as `/geyser/key.pem`. Each backend mounts it at `/data/plugins/floodgate/key.pem`. If it is lost, generate a new one (`openssl rand -out /opt/deablr/floodgate/key.pem 16`) and restart Geyser plus all backends.

## Requirements

- Hostinger VPS with Docker and Docker Compose installed
- [Dokploy](https://dokploy.com) installed on the server
- Both backend servers deployed as separate Dokploy Compose projects
- DNS `A` records for `richard.deablr.com` and `nijika.deablr.com` pointing to the VPS public IP

## Backend Server Requirements

For the router to reach the backend servers, each server Compose project **must**:

1. Be attached to the `dokploy-network` network:

   ```yaml
   services:
     mc:
       # ...
       networks:
         - dokploy-network

   networks:
     dokploy-network:
       external: true
   ```

2. **Not** bind host port `25565` themselves. Only `mc-router` binds the public `25565` port. Remove or comment out:

   ```yaml
   # Do NOT include this in the backend servers
   ports:
     - "25565:25565"
   ```

The backend containers will still listen on container port `25565`; only the host binding is removed.

3. **Allow Bedrock players** (optional): install the [Floodgate](https://geysermc.org/wiki/floodgate/) plugin and mount the shared key so Geyser's translations are trusted:

   ```yaml
   services:
     mc:
       environment:
         PLUGINS: |-
           https://download.geysermc.org/v2/projects/floodgate/versions/latest/builds/latest/downloads/spigot
       volumes:
         - "/opt/deablr/floodgate/key.pem:/data/plugins/floodgate/key.pem:ro"
   ```

## Deploying with Dokploy

1. In Dokploy, create a new **Compose** project.
2. Connect this Git repository (`Deablr/mc-router-deablr`).
3. Dokploy will run `docker compose up -d` and start `mc-router` and `geyser`.
4. Ensure both backend servers are also running in Dokploy on the same `dokploy-network`.
5. Open **UDP 19132** on the VPS firewall (and in Hostinger's panel firewall, if enabled), otherwise Bedrock players get "Unable to connect to world".

## DNS Setup

Create `A` records in your DNS provider pointing both domains to the same VPS IP:

```
richard.deablr.com  A  <VPS_IP>
nijika.deablr.com   A  <VPS_IP>
```

No `SRV` records are needed because both servers use the default Minecraft port.

## Configuration

Edit `compose.yml` to add or remove routed servers:

```yaml
environment:
  MAPPING: |
    richard.deablr.com=richard-deablr-mc:25565
    nijika.deablr.com=nijika-deablr-mc:25565
```

Use the backend container's `container_name` as the hostname to avoid conflicts with the default service name `mc` used by both projects.

## Ports

- `25565` — Minecraft router (default Minecraft port, Java Edition)
- `19132/udp` — Geyser's Bedrock listener (default Bedrock port)

## File Structure

```
.
├── compose.yml   # Docker Compose configuration for mc-router and geyser
├── geyser/
│   ├── Dockerfile   # Geyser Standalone image (pinned build; bump URL to upgrade)
│   └── config.yml   # Geyser configuration
└── README.md     # This file
```

## Troubleshooting

### `bind: address already in use` on port 25565

Another container or backend server is already bound to host port `25565`. Stop the conflicting service and ensure only `mc-router` exposes `25565` on the host.

### Connection times out

- Verify DNS `A` records point to the VPS IP.
- Confirm both backend containers are running and attached to `dokploy-network`.
- Check that the backend container names in `MAPPING` match the actual `container_name` values.

### Wrong server loads

The Minecraft client may cache the resolved server. Restart the client or flush DNS to ensure the new domain mapping is used.

### Bedrock player gets "Unable to connect to world" immediately

**UDP 19132** is not open on the VPS or the provider firewall, or the `geyser` container is not running (`docker logs geyser-deablr`).

### Bedrock player gets "An internal error happened while handling Floodgate data"

Geyser and the backend have different `key.pem` files, or the backend lacks Floodgate. Check the key mount on both sides (shared key: `/opt/deablr/floodgate/key.pem`).

### Bedrock player disconnects and no backend log entry appears

The hostname has no `MAPPING` entry in `mc-router` — same as an unrouted Java join.

### `Failed to ping the remote Java server!` spam in Geyser logs

`motd.passthrough-motd` / `motd.passthrough-player-counts` must be disabled — a Bedrock server-list ping carries no hostname, so `mc-router` cannot route it.

### Player lands on the wrong server

DNS/hostname mismatch; check the `server=` field in `docker logs mc-router-deablr`.

## Notes

- `mc-router` does **not** use Traefik; it handles Minecraft protocol routing directly.
- Keep backend server host port mappings disabled so they do not conflict with `mc-router`.
- The Bedrock server list shows one fixed MOTD/player count for all hostnames: a Bedrock ping cannot be routed by hostname, so Geyser serves its own MOTD (`bedrock.server-name` / `motd.*` in `geyser/config.yml`) instead of relaying each backend's.
