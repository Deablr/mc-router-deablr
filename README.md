# Minecraft Router — Deablr

SNI-based Minecraft router that lets multiple Deablr Minecraft servers share a single public IP and the standard Minecraft port (`25565`).

Players connect to the router using a domain name, and `itzg/mc-router` forwards the connection to the correct backend server.

## Routed Servers

| Domain | Backend Container | Description |
|--------|-------------------|-------------|
| `richard.deablr.com` | `richard-deablr-mc:25565` | Richard's creative flat Paper `26.1.2` server |
| `nijika.deablr.com` | `nijika-deablr-mc:25565` | Nijika's Paper `26.1.2` server |

## How It Works

1. `mc-router` listens on host port `25565`.
2. A Minecraft client connects to `richard.deablr.com:25565` or `nijika.deablr.com:25565`.
3. `mc-router` reads the Server Name Indication (SNI) hostname and proxies the connection to the matching backend container.
4. The backend containers are reached over the shared `deabcraft` Docker network.

## Requirements

- Hostinger VPS with Docker and Docker Compose installed
- [Dokploy](https://dokploy.com) installed on the server
- Both backend servers deployed as separate Dokploy Compose projects
- DNS `A` records for `richard.deablr.com` and `nijika.deablr.com` pointing to the VPS public IP

## Backend Server Requirements

For the router to reach the backend servers, each server Compose project **must**:

1. Be attached to the `deabcraft` network:

   ```yaml
   services:
     mc:
       # ...
       networks:
         - deabcraft

   networks:
     deabcraft:
       external: true
   ```

2. **Not** bind host port `25565` themselves. Only `mc-router` binds the public `25565` port. Remove or comment out:

   ```yaml
   # Do NOT include this in the backend servers
   ports:
     - "25565:25565"
   ```

The backend containers will still listen on container port `25565`; only the host binding is removed.

## Deploying with Dokploy

1. In Dokploy, create a new **Compose** project.
2. Connect this Git repository (`Deablr/mc-router-deablr`).
3. Dokploy will run `docker compose up -d` and start `mc-router`.
4. Ensure both backend servers are also running in Dokploy on the same `deabcraft`.

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

- `25565` — Minecraft router (default Minecraft port)

## File Structure

```
.
├── compose.yml   # Docker Compose configuration for mc-router
└── README.md     # This file
```

## Troubleshooting

### `bind: address already in use` on port 25565

Another container or backend server is already bound to host port `25565`. Stop the conflicting service and ensure only `mc-router` exposes `25565` on the host.

### Connection times out

- Verify DNS `A` records point to the VPS IP.
- Confirm both backend containers are running and attached to `deabcraft`.
- Check that the backend container names in `MAPPING` match the actual `container_name` values.

### Wrong server loads

The Minecraft client may cache the resolved server. Restart the client or flush DNS to ensure the new domain mapping is used.

## Notes

- `mc-router` does **not** use Traefik; it handles Minecraft protocol routing directly.
- Keep backend server host port mappings disabled so they do not conflict with `mc-router`.
