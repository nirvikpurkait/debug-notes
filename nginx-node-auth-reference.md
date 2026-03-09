# Nginx Reverse Proxy Config for Node.js Apps with Auth

### Reference Guide — Works with Better Auth, NextAuth, Supabase Auth, etc.

---

## The Problem This Solves

When you run a Node.js app behind Nginx on a VPS, three things silently break authentication:

| #   | Problem                                     | Root Cause                                        | Symptom                                                                                                            |
| --- | ------------------------------------------- | ------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| 1   | Nginx doesn't tell your app it's on HTTPS   | Missing `X-Forwarded-Proto` header                | Auth sets insecure cookies → browser drops them → state mismatch / login loop                                      |
| 2   | Nginx doesn't tell your app the real domain | Missing `X-Forwarded-Host` header                 | Auth builds wrong callback URLs like `http://localhost:3000/callback` instead of `https://yourdomain.com/callback` |
| 3   | Nginx default header buffer is only 8kb     | JWT/session cookies in response headers are large | `upstream sent too big header` → 502 Bad Gateway on login/callback                                                 |

All three cause **502 errors on sign-in** even though the rest of your app works fine.
The misleading part: your Docker logs may show `state mismatch` errors — those are _secondary_ errors caused by the real failures above.

---

## How Each Fix Works

### Fix 1 — `X-Forwarded-Proto https`

Nginx sits between the browser (HTTPS) and your app (plain HTTP internally).
Your app only sees the internal HTTP connection and thinks it's insecure.
Auth libraries use the protocol to decide cookie flags — if they think it's HTTP,
they won't set `Secure` on cookies. Browsers silently drop non-Secure cookies over HTTPS.
This header tells your app "the original connection was HTTPS, act accordingly."

### Fix 2 — `X-Forwarded-Host $host`

Your app receives requests from `127.0.0.1`, not `yourdomain.com`.
When auth libraries build redirect/callback URLs, they read the host from the request.
Without this header, they build `http://127.0.0.1:3000/api/auth/callback/google` —
which Google rejects, and which doesn't match what your auth config expects.
This header passes the real public domain through the proxy.

### Fix 3 — `proxy_buffer_size 128k` + `proxy_buffers`

After a successful OAuth callback, your auth library sets a session cookie in the
response headers. JWT-based sessions encode user data, tokens, and signatures —
easily 2–10kb just for headers. Nginx's default buffer is 8kb total.
When headers exceed this, Nginx aborts with a 502 and logs:
`upstream sent too big header while reading response header from upstream`
Increasing the buffer to 128k–256k gives headers room to breathe.

---

## The Config

```nginx
# ============================================================
# BLOCK 1: Redirect all HTTP → HTTPS
# Catches port 80 traffic and permanently redirects to HTTPS.
# $request_uri preserves the path e.g. /about, /login
# ============================================================
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;

    return 301 https://yourdomain.com$request_uri;
}


# ============================================================
# BLOCK 2: Redirect HTTPS www → HTTPS non-www
# Canonicalizes your domain so www and non-www aren't
# treated as separate sites by browsers and search engines.
# ============================================================
server {
    listen 443 ssl http2;
    server_name www.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    return 301 https://yourdomain.com$request_uri;
}


# ============================================================
# BLOCK 3: Main HTTPS server — proxy traffic to your app
# All real traffic lands here after the redirects above.
# ============================================================
server {
    listen 443 ssl http2;
    server_name yourdomain.com;

    # SSL certs from Let's Encrypt / Certbot
    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    # Only allow modern TLS — v1.0 and v1.1 are deprecated and insecure
    ssl_protocols TLSv1.2 TLSv1.3;

    location / {

        # ----- Core Proxy -----
        proxy_pass http://127.0.0.1:5000;          # Your app's host port (Docker maps container → host)
        proxy_http_version 1.1;                    # HTTP/1.1 required for keep-alive connections


        # ----- Forward Headers: Tell your app about the real request -----
        # Without these, your app thinks every request is plain HTTP from localhost.
        # Auth libraries break silently without this information.

        proxy_set_header Host $host;
        # Passes yourdomain.com to the app.
        # Without it: app sees 127.0.0.1 as the host.

        proxy_set_header X-Real-IP $remote_addr;
        # Passes the real visitor IP.
        # Without it: app sees Nginx's own IP for every request.

        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        # Full chain of IPs the request passed through.
        # Used for logging, rate limiting, fraud detection.

        proxy_set_header X-Forwarded-Proto https;
        # *** FIX #1 ***
        # Tells the app the original protocol was HTTPS.
        # Auth libraries use this to set Secure cookie flags.
        # Without it: cookies are insecure → browser drops them → 502 on login.

        proxy_set_header X-Forwarded-Port 443;
        # Tells the app the original port.
        # Some auth libraries use this when building redirect URLs.

        proxy_set_header X-Forwarded-Host $host;
        # *** FIX #2 ***
        # Tells the app the original public domain.
        # Auth libraries use this to build callback URLs.
        # Without it: callback URL points to localhost → OAuth fails.


        # ----- Buffer Sizes: Fix 502 on login with JWT/session auth -----
        # *** FIX #3 ***
        # Default Nginx buffer is 8kb. JWT session cookies easily exceed this.
        # Symptom in logs: "upstream sent too big header while reading response header"
        # Applies to: Better Auth, NextAuth, Supabase Auth, Passport, Auth.js, etc.

        proxy_buffer_size       128k;              # Buffer for response headers (first part of response)
        proxy_buffers           4 256k;            # Number × size of buffers for the response body
        proxy_busy_buffers_size 256k;              # Max buffer size actively being sent to client


        # ----- Timeouts: Prevent 502 on slow OAuth round-trips -----
        # OAuth flows make external calls to Google/GitHub/etc. which can be slow.
        # Without these, Nginx gives up before the auth provider responds.

        proxy_read_timeout    60s;                 # Wait up to 60s for app to send a response
        proxy_connect_timeout 60s;                 # Wait up to 60s to connect to the app
        proxy_send_timeout    60s;                 # Wait up to 60s while sending a request to the app


        proxy_redirect off;                        # Don't rewrite Location headers from the app
                                                   # App already sends correct absolute URLs
    }
}
```

---

## Quick Checklist — Every Time You Deploy a New App

- [ ] Replace `yourdomain.com` with your actual domain (all 4 occurrences)
- [ ] Replace port `5000` in `proxy_pass` with your Docker host port mapping
- [ ] SSL certs exist at the Let's Encrypt path (run `certbot` if not)
- [ ] Your app's auth config has `baseURL` set to `https://yourdomain.com`
- [ ] Your app's auth config has `secret` explicitly set (not relying on auto-generated)
- [ ] `useSecureCookies: true` in your auth config (Better Auth / NextAuth)
- [ ] Google OAuth Console has `https://yourdomain.com/api/auth/callback/google` registered
- [ ] After editing: run `nginx -t && systemctl reload nginx`

---

## Common Nginx Errors and What They Mean

| Nginx Log Error                                   | Actual Cause                                    | Fix                                                 |
| ------------------------------------------------- | ----------------------------------------------- | --------------------------------------------------- |
| `upstream sent too big header`                    | JWT/session cookie too large for default buffer | Add `proxy_buffer_size 128k` + `proxy_buffers`      |
| `connect() failed (111: Connection refused)`      | App isn't running or wrong port in `proxy_pass` | Check `docker ps`, verify port mapping              |
| `upstream timed out`                              | App took too long to respond                    | Increase `proxy_read_timeout`                       |
| `SSL_do_handshake() failed`                       | Bad/expired SSL cert                            | Run `certbot renew`                                 |
| `state mismatch / cookie not found` (in app logs) | Usually caused by one of the above failures     | Fix the Nginx error first, app errors are secondary |

---

## Why This Works Locally But Not in Production

| Factor             | Local Dev                            | Production                                           |
| ------------------ | ------------------------------------ | ---------------------------------------------------- |
| Protocol           | `http://` — no Secure cookie rules   | `https://` — browser enforces Secure flag            |
| Proxy hops         | 0 — browser talks directly to app    | 1 — Nginx sits in between                            |
| Cookie Secure flag | Not required                         | Required — browser silently drops cookies without it |
| Header buffer      | N/A — no proxy                       | Nginx's 8kb default gets exceeded by JWT cookies     |
| baseURL detection  | App reads `localhost:3000` correctly | App reads `127.0.0.1` through proxy — wrong          |

---

_Tested with: Better Auth · NextAuth · Supabase Auth on Node.js + Nitro + Docker + Nginx + VPS_
