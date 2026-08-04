# Main nginx (reverse proxy)

The single public entry point for the server. Routes:

- `crossword.yehor-inq.com` → `crossword-nginx` (Django/gunicorn, via the crossword project's own nginx container)
- `splitbot.yehor-inq.com` → `splitbot-app` (FastAPI/uvicorn web-auth service)

Both are reached over a shared external Docker network called `edge` — this
proxy never talks to app containers over the host network, and the app
stacks don't publish any ports to the host themselves.

DNS for both subdomains is proxied through Cloudflare (orange cloud), with
SSL/TLS mode **Full / Full strict**. That means Cloudflare terminates TLS
for visitors, then re-encrypts and connects to this server on port 443 —
so this nginx needs its own certificate for that second hop. Since
Cloudflare doesn't validate that cert against a public CA in Full mode (and
does validate it in Full *strict*), the right tool here is a **Cloudflare
Origin CA certificate**, not Let's Encrypt — no certbot, no renewal cron,
valid for up to 15 years.

## One-time server setup

### 1. Create the shared network

```bash
docker network create edge
```

### 2. Get a Cloudflare Origin CA certificate

In the Cloudflare dashboard: **SSL/TLS → Origin Server → Create Certificate**.

- Hostnames: `*.yehor-inq.com, yehor-inq.com` (one cert covers both subdomains)
- Key type: RSA (2048) is fine
- Validity: 15 years

Save the two values it gives you on the server as:

```
nginx-proxy/certs/origin.pem   # "Origin Certificate"
nginx-proxy/certs/origin.key   # "Private Key" — shown once, save it now
```

```bash
mkdir -p certs
chmod 600 certs/origin.key
```

### 3. Make sure Cloudflare SSL/TLS mode is Full or Full (strict)

**Flexible** must *not* be used — it would make Cloudflare connect to this
origin over plain HTTP, and the conf files here only listen on 443 for the
actual app traffic (port 80 is just an HTTPS redirect safety net).

### 4. Bring everything up, in order

```bash
# 1. app stacks first, so the proxy has something to reach
cd ../crossword && docker compose up -d --build
cd ../splitbot  && docker compose up -d --build

# 2. then the proxy
cd ../nginx-proxy && docker compose up -d
```

### 5. Verify

```bash
docker compose logs -f nginx
curl -H "Host: crossword.yehor-inq.com" http://127.0.0.1
curl -H "Host: splitbot.yehor-inq.com" http://127.0.0.1
```

Then check both sites through the real domains (with Cloudflare proxying).

## Optional hardening: firewall origin to Cloudflare IPs only

Since Cloudflare's Origin CA cert is only meant to be trusted by Cloudflare,
it's worth making sure nothing else can reach port 443/80 directly by IP,
bypassing Cloudflare (and its WAF/rate limiting). On the host firewall
(e.g. `ufw`), allow 80/443 only from Cloudflare's published ranges
(https://www.cloudflare.com/ips/) instead of `0.0.0.0/0`. `conf.d/00-cloudflare-realip.conf`
already restores the real visitor IP from `CF-Connecting-IP` for logs/`X-Forwarded-For`,
regardless of whether you add the firewall rule.

## Adding a third app later

1. Give it its own `docker-compose.yml` with no host port publish, joined to
   the external `edge` network, `container_name` set to something DNS-safe.
2. Drop a new `conf.d/<name>.conf` here following the pattern in
   `crossword.conf` / `splitbot.conf`, pointing `proxy_pass` at that
   container name.
3. `docker compose restart nginx` (or `nginx -s reload` inside the container).
