# Personal site

A one-page site served by nginx in a small Docker container.

## Before you build

1. Open `site/index.html` and replace the placeholders:
   - `Your Name` (it appears twice in the name heading, plus once in the photo label and the page title)
   - `you@example.com` (email link)
   - `https://www.linkedin.com/in/your-profile` (LinkedIn link)
2. Put your résumé in the `site/` folder and name it `resume.pdf`.

## Run it

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

After changing anything in `site/`, rebuild and restart:

```
docker compose up -d --build
```

## Put it online

The container serves plain HTTP on port 8080. To publish it on a domain, run it on a server or
hosting service that supports containers and put HTTPS in front of it (a reverse proxy such as
Caddy or Traefik, or your host's built-in HTTPS).
