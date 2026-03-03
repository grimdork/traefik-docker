# Traefik gateway for Docker/Podman
A production-ready Traefik deployment for small servers, featuring ACME resolvers (DNS-01 via Scaleway and HTTP-01).

## Overview
This configuration acts as the entry point for all web traffic. It handles:
- Certificates via Let's Encrypt (Scaleway DNS and HTTP challenges).
- Header obfuscation to hide server identity.
- Dynamic routing: Automatically detects Docker containers with specific labels.

Just make a DNS record and run your simple web-facing services via Docker compose files.

## Requirements
Linux server with ACL support and basic command line tools, ssh access and Docker or Podman (or anything else compatible with docker-compose files).

## Files
- docker-compose.yml: The core service definition.
- .env: Required environment variables go here.
- grant-docker: Utility script to manage Docker directory permissions via Linux ACLs.
- web/: An example of a simple static web server which can be used as many times as the server can handle.
- - docker-compose.yml: Example of an nginx compose file.
- - .env: Edit this for each web server to give them unique names.

## Setup & installation

### Copy the repo and make Let's encrypt files
Copy this directory to wherever you want to set up a new traefik configuration. Remove the .git folder from the copy.

Example
```bash
mkdir /home/docker
cp -r traefik-docker /home/docker/traefik
cd /home/docker/traefik
rm -rf .git
mkdir letsencrypt
touch letsencrypt/acme.json letsencrypt/acme-http.json
```

Then copy the web/ directory to other places, or delete it.

### Convenience

Create the docker group if it doesn't exist:
```bash
sudo groupadd docker
```

Add a user to the docker group:
```bash
sudo usermod -aG docker <username>
```

Now that user can run docker without sudo.

### Prerequisites
Ensure you have a Docker network named proxy-net created:

```bash
docker network create proxy-net
```

### Environment Variables
Edit .env and fill in your credentials:
- SCW_...: Your Scaleway API keys for DNS challenges.
- ACME_EMAIL: Your contact email for Let's Encrypt.
- GATEWAY_DOMAIN: The name you want to appear in the Server header.

### The grant-docker script
Because the proxied containers behind Traefik often require persistent storage, use the provided script to ensure containers have the correct access to your local directories:

```bash
chmod +x grant-docker
./grant-docker <path>
```

This allows you to give Docker access to user directories owned by them. You can find more information in the `setfacl` documentation.

NOTE: If you move these directories or restore from a backup that doesn't support extended attributes, you may need to re-run `grant-docker`.

### File Preparation
Ensure your ACME storage files have the strict permissions required by Traefik:

```bash
chmod 600 letsencrypt/*.json
```

## Security Features
### Header Obfuscation
This gateway includes a hide-identity middleware. Instead of revealing the server version, it returns:

```
Server: ${GATEWAY_DOMAIN} (from your .env)
X-Powered-By: Chaos
````

## Resolvers

|Resolver|Type|Use case|
|--------|----|--------|
|scw|DNS-01|Set up via Scaleway hosted DNS zones|
|acmeweb|HTTP-01|Handle the ACME response via HTTP|

NOTE: The first certificate issued may take a couple of minutes.

### Self-signed certificate
This gateway uses an internal certificate initially. Create it like this while in the configuration folder:
```bash
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -sha256 -days 3650 -nodes -subj "/CN=localhost"
```

Any domains you configure in other containers will get ACME certificates.

## Usage in Other Containers
To route a service through this gateway, add these labels to your application's docker-compose.yml:

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.myapp.rule=Host(`example.com`)" # Replace with your domain
  - "traefik.http.routers.myapp.entrypoints=websecure"
  - "traefik.http.routers.myapp.tls.certresolver=scw" # Use scw or acmeweb
  - "traefik.http.services.myapp.loadbalancer.server.port=80" # Just to avoid potential headaches, put the exposed port of your app here
```

Replace `myapp` with a unique identifier, preferably via environment variables, and likewise for the domain. See the included compose files for fuller examples.

## Troubleshooting

### Visual Studio Code users
You may have RedHat's highlighter installed for YAML, and it may be set to format on save. This strips out backslashes, which are vital to Docker compose files. Disable it to avoid trouble.

### 404 Not Found
- Verify the container is joined to the `proxy-net` network.
- Check that the `traefik.http.services.myapp.loadbalancer.server.port` matches the port the container is listening on internally.

### Certificate stays Self-Signed
- Check Traefik logs: `docker logs -f traefik`.
- Ensure your `.env` variables for Scaleway are correct.
- DNS-01 challenges can take up to 2-5 minutes to propagate.

### 429 error
There's an error, like DNS not having propagated yet, and you're hammering Let's Encrypt too fast. Run `docker compose down` and wait 5 minutes.

Tip: While testing, you can use the staging server by adding `--certificatesresolvers.scw.acme.caserver=https://acme-staging-v02.api.letsencrypt.org/directory` to your Traefik commands to avoid hitting production rate limits. This will issue a certificate named "Fake LE Intermediate X1" which will give browser warnings, but you won't be throttled for many attempts.

### Permission Denied (Volumes)
- If a container cannot write to its `/data` directory, re-run:
  `./grant-docker /path/to/your/app/data`

### Can't access the site
Ensure you have allowed firewall access to ports 80 and 443. If you're using a VPS with some automated setup, you may have a Linux distro with UFW set up, in which case pretty much everything but SSH is closed off. Allow the ports like this:
```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```

or using an app definition:
```bash
sudo ufw allow "Nginx Full"
# OR
sudo ufw allow "Apache Full"
```
These are just shorthand for the web ports, so having the actual servers isn't required.

Check status after:
```bash
sudo ufw status numbered
```

You may also have a VPS/server with an extra firewall layer outside, like Hetzner VPSes. Check their control panel. AWS VPSes (compute servers) typically have the same issue as UFW, and you need to explicitly configure security groups to allow the web services.
