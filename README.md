# mcpproxy-docker

Unofficial Docker builds of [smart-mcp-proxy/mcpproxy-go](https://github.com/smart-mcp-proxy/mcpproxy-go). Upstream ships a Dockerfile but doesn't publish images, so this repo builds and pushes one to GHCR whenever they cut a new release.

## Image

```
ghcr.io/andrewpaglusch/mcpproxy
```

Tags follow upstream's `vMAJOR.MINOR.PATCH` scheme (the `v` is dropped):

- `latest` (most recent upstream release)
- `0` (latest 0.x)
- `0.33` (latest 0.33.x)
- `0.33.5` (specific release)

## Usage

`docker-compose.yml`:

```yaml
services:
  mcpproxy:
    image: ghcr.io/andrewpaglusch/mcpproxy:latest
    container_name: mcpproxy
    restart: unless-stopped
    command: ["--config", "/data/mcp_config.json"]
    volumes:
      - ./mcp_config.json:/data/mcp_config.json:ro
      - ./data:/data
    networks: [proxy]
    labels:
      - traefik.enable=true
      - traefik.docker.network=proxy
      - traefik.http.services.mcpproxy.loadbalancer.server.port=8080
      # /mcp: agents authenticate with the API key / a token, no basic auth
      - traefik.http.routers.mcpproxy-mcp.rule=Host(`mcpproxy.example.com`) && PathPrefix(`/mcp`)
      - traefik.http.routers.mcpproxy-mcp.entrypoints=websecure
      - traefik.http.routers.mcpproxy-mcp.tls.certresolver=le
      - traefik.http.routers.mcpproxy-mcp.service=mcpproxy
      # UI: basic auth for humans
      - traefik.http.routers.mcpproxy-ui.rule=Host(`mcpproxy.example.com`)
      - traefik.http.routers.mcpproxy-ui.entrypoints=websecure
      - traefik.http.routers.mcpproxy-ui.tls.certresolver=le
      - traefik.http.routers.mcpproxy-ui.service=mcpproxy
      - traefik.http.routers.mcpproxy-ui.middlewares=mcpproxy-auth
      - traefik.http.middlewares.mcpproxy-auth.basicauth.users=admin:$$apr1$$abcd1234$$0123456789abcdef0123456

networks:
  proxy:
    external: true
```

`mcp_config.json`:

```json
{
  "listen": "0.0.0.0:8080",
  "data_dir": "/data",
  "routing_mode": "retrieve_tools",
  "require_mcp_auth": true,
  "api_key": "your-secret-key",
  "mcpServers": [
    {
      "name": "example",
      "protocol": "streamable-http",
      "url": "https://example-mcp.your-domain.com/mcp",
      "headers": {
        "Authorization": "Bearer your-upstream-token"
      },
      "enabled": true
    }
  ]
}
```

## How it works

A GitHub Actions workflow runs daily at 06:00 UTC. It checks the latest release on the upstream repo, and if there isn't already an image in GHCR for that tag, it clones upstream at the tag, builds the Dockerfile, and pushes.

Right now only `linux/amd64` is built.

## Manual builds

You can trigger a build from the Actions tab. There are two optional inputs:

- `tag`: build a specific upstream release tag instead of the latest one
- `force`: rebuild even if the image tag already exists in GHCR

## Notes

I don't maintain the proxy itself. For bugs, features, or anything about how it actually works, please go to the [upstream repo](https://github.com/smart-mcp-proxy/mcpproxy-go). Issues here should be limited to packaging problems (build failures, missing tags, etc).
