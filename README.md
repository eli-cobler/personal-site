# Personal site

A one-page site served by nginx in a small Docker container.

## Before you build

1. Open `site/index.html` and replace the remaining placeholders:
   - `Your Name` (it appears twice: once in the page `<title>`, once as the name heading that types itself out)
   - `you@example.com` (email link)
   - `https://www.linkedin.com/in/your-profile` (LinkedIn link)
2. Put your résumé in the `site/` folder and name it `resume.pdf`.
3. To change the photo later, replace `site/photo.jpg` with a new image of the same name (or update the
   `src="photo.jpg"` in `index.html` if you use a different filename or format, like `.png`).

## Run it (single platform, e.g. just testing locally)

```
docker build -t personal-site .
docker run -d --name personal-site -p 8080:8080 personal-site
```

Then open http://localhost:8080.

Or with Docker Compose:

```
docker compose up -d --build
```

## Update it

After changing anything in `site/` — including swapping the photo or résumé — rebuild and restart:

```
docker compose up -d --build
```

## Building for multiple CPU architectures (e.g. Apple Silicon + x86 servers)

A build made on an Apple Silicon (ARM64) Mac only runs on ARM64 machines by default. To produce an
image that runs on both ARM64 and x86_64 (amd64) machines, use Docker's `buildx` tool. No changes
to the Dockerfile are needed — this site has no architecture-specific dependencies.

1. **Check buildx is available:**
   ```
   docker buildx version
   ```
   Recent Docker Desktop versions include it by default.

2. **Create and switch to a multi-platform builder (one-time setup):**
   ```
   docker buildx create --name multiplat --use
   docker buildx inspect --bootstrap
   ```
   `--bootstrap` starts the builder and sets up emulation so an ARM Mac can also produce amd64 layers.

3. **Log in to a registry** (Docker Hub, GHCR, etc.) if you haven't already:
   ```
   docker login
   ```

4. **Build for both platforms and push in one command:**
   ```
   docker buildx build --platform linux/amd64,linux/arm64 -t YOUR_USERNAME/personal-site:latest --push .
   ```
   The trailing `.` (the build context, meaning "this folder") is required — leaving it off is what
   causes the `requires 1 argument` error. Run this from inside the `personal-site` folder, where the
   `Dockerfile` lives.

   Multi-platform builds must be pushed to a registry with `--push`; buildx can't load a multi-platform
   image straight into your local Docker. To just test the amd64 build locally on your Mac without
   pushing anywhere, build one platform at a time instead:
   ```
   docker buildx build --platform linux/amd64 -t personal-site:amd64 --load .
   ```

5. **Run it anywhere:**
   ```
   docker pull YOUR_USERNAME/personal-site:latest
   docker run -d -p 8080:8080 YOUR_USERNAME/personal-site:latest
   ```
   Docker automatically pulls the layer that matches whatever machine you run this on.

## Put it online with a domain, via Cloudflare Tunnel

`docker-compose.yml` already includes a `cloudflared` service. It makes an outbound-only
connection from your server to Cloudflare, so your site gets a real domain and free HTTPS
without opening any ports on your server's firewall. This needs to run somewhere with internet
access — a VPS, home server, Raspberry Pi, etc. (not this sandbox).

1. **Point your domain at Cloudflare**, if it isn't already (Cloudflare dashboard > your domain >
   Overview > follow the nameserver instructions).

2. **Create a tunnel:** Cloudflare dashboard > Zero Trust > Networks > Tunnels > Create a tunnel >
   Cloudflared. Name it (e.g. `personal-site`).

3. **Set the public hostname**, still in that tunnel's setup: enter your subdomain (e.g.
   `site.yourdomain.com`), service type **HTTP**, and service address **`site:8080`** — `site` is
   the name of the container in `docker-compose.yml`, and Docker resolves it automatically since
   both containers share the `internal` network. This step also creates the DNS record for you —
   no manual entry needed.

4. **Copy the tunnel token** shown on the same page, then on your server:
   ```
   cp .env.example .env
   ```
   and paste the token in as `TUNNEL_TOKEN=...` in `.env`. Keep this file private — anyone with the
   token can route traffic through your tunnel. It's already excluded from the Docker build via
   `.dockerignore`.

5. **Start everything:**
   ```
   docker compose up -d --build
   ```
   In the Cloudflare dashboard, the tunnel should switch to "Healthy" within a few seconds.

6. **Visit `https://site.yourdomain.com`.** It should load your site with a valid certificate,
   issued and renewed automatically by Cloudflare.

Because the `site` service no longer publishes a port to the host (only `cloudflared` can reach it,
over the internal Docker network), nothing here needs port 80 or 443 open, and http://localhost:8080
from the "Run it" section above will stop working once you remove that port mapping — that's expected
once cloudflared is the only way in.

### Alternative: your own reverse proxy

If you'd rather not use Cloudflare Tunnel, you can instead add an A record in Cloudflare's DNS tab
pointing at your server's public IP (orange-cloud on for edge HTTPS), and run a reverse proxy such
as Caddy in front of this site's container, which can fetch its own Let's Encrypt certificate
automatically. That approach needs ports 80/443 open on your server, and more of your own upkeep,
but gives you more control if you're already running other services on that machine.
