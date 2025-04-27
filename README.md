# opensign-traefik
This repository provides a complete Docker Compose setup to run [OpenSign](https://www.opensignlabs.com/) behind a [Traefik](https://traefik.io/traefik/) reverse proxy. It enables a simple, secure, and production-ready deployment of OpenSign with automatic routing, HTTPS (Let's Encrypt), and configurable domain settings.

I noticed on the [OpenSign Discord](https://discord.com/invite/xe9TDuyAyj) that many users are having trouble hosting OpenSign behind a Traefik reverse proxy. After reviewing the discussions, I realized that most people are simply forgetting to include the following lines in their docker-compose.yml:
```
- "traefik.http.routers.server.middlewares=server-strip-api@docker"
- "traefik.http.middlewares.server-strip-api.stripprefix.prefixes=/api"
```
These lines are crucial for properly stripping the /api prefix, ensuring that Traefik routes the requests correctly to OpenSign.

To help others, I’ve created a working example configuration that includes these settings, making it easier to deploy OpenSign behind Traefik without issues.
