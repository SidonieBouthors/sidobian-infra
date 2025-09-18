# Sidobian Server Configuration

This repository contains the configuration of my personal server. 
It uses [Docker Compose](https://docs.docker.com/compose/) for services 
and a [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/) to route traffic and serve as a reverse proxy.

Each service has a dedicated `docker-compose.yml`, and all use the Docker network called `cloudflare-network` rather than binding to ports on the host. 
Linking each service to a public URL is done directly from the Cloudflare Zero Trust dashboard.
