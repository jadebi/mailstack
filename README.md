# mailstack (docker)

A Docker-based mail server and webmail client stack using [Stalwart](https://stalw.art) and [Bulwark](https://bulwarkmail.org), fronted by [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/).

> **Please note:** This repo wires the three services together. If you run into issues with Stalwart, Bulwark or Cloudflare itself, head over to their docs or issue trackers (links above). GitHub issues here should be about the compose setup only.

This branch (`tiny-vps`) adds resource limits for a ~1 vCPU, ~2 GB RAM VPS. For the base config without limits, switch to the [`main`](https://github.com/jadebi/mailstack/tree/main) branch.

## Stack

- **Stalwart:** mail server (SMTP, IMAP, JMAP, antivirus, spam filtering). [ [Docs](https://stalw.art/docs) | [GitHub](https://github.com/stalwartlabs) ]
- **Bulwark:** webmail client. [ [Docs](https://bulwarkmail.org/docs) | [GitHub](https://github.com/bulwarkmail) ]
- **Cloudflare Tunnel:** secure outbound-only connectivity with no open firewall ports. [ [Docs](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/) ]

## Getting Started

1. Clone the repository
  ```sh
  git clone https://github.com/jadebi/mailstack.git --branch tiny-vps --single-branch
  cd mailstack
  ```
2. Create data folders with the correct permissions
  ```sh
  mkdir -p \
    stalwart/etc \
    stalwart/data \
    bulwark/settings \
    bulwark/admin \
    bulwark/admin-state \
    bulwark/telemetry
    
  sudo chown -R 2000:2000  ./stalwart
  sudo chown -R 1001:65533 ./bulwark
  ```
3. Copy the example env files
  ```sh
   cp .env.bulwark.example .env.bulwark
   cp .env.tunnel.example .env.tunnel
  ```
4. Edit `.env.bulwark` and set `JMAP_SERVER_URL` to your Stalwart JMAP endpoint (e.g. `https://mail.example.com`).
5. Set up a Cloudflare Tunnel (see below) and paste the token into `.env.tunnel`.
6. Start everything:
  ```sh
   docker compose up -d
  ```

Bulwark will be available at `webmail.example.com` and Stalwart at `mail.example.com` (or whatever domains you route through the tunnel).

## Without Cloudflare Tunnel

If you prefer to expose services directly (or already have a reverse proxy), you need to:

1. Uncomment the HTTP ports in `[docker-compose.yml](./docker-compose.yml)`:
  - Stalwart: ports `443:443` and `8080:8080`
  - Bulwark: port `3000:3000`
2. Point DNS A records for `mail.example.com` and `webmail.example.com` to your server's public IP.
3. Set up your own reverse proxy with TLS certificates (e.g. Caddy, Nginx + Let's Encrypt) on ports 443 and 80.
4. Make sure the Stalwart mail ports are reachable (see below).

The Cloudflare tunnel service and its `.env.tunnel` file can be left alone or removed from the compose file.

## Port Forwarding

Stalwart needs these ports open for mail delivery and client access:  
*You also need to forward these ports if you use Cloudflare tunnels!*

| Port | Protocol | Purpose                    |
| ---- | -------- | -------------------------- |
| 25   | SMTP     | Incoming mail              |
| 587  | SMTP     | Mail submission            |
| 465  | SMTP     | Submission over TLS        |
| 143  | IMAP     | IMAP (plain/StartTLS)      |
| 993  | IMAP     | IMAP over TLS              |
| 110  | POP3     | POP3 (plain/StartTLS)      |
| 995  | POP3     | POP3 over TLS              |
| 4190 | JMAP     | JMAP API (used by Bulwark) |

Forward these ports on your router/firewall to the internal IP of the machine running the stack. Some ISPs block port 25 (SMTP) on residential connections. You may need a business plan or a relay service if that is the case.

If you are **not** using Cloudflare Tunnel, you also need to forward the web interface ports:

| Port | Service  | Purpose                      |
| ---- | -------- | ---------------------------- |
| 443  | Stalwart | HTTPS (JMAP API, management) |
| 8080 | Stalwart | HTTP (With reverse proxy)    |
| 3000 | Bulwark  | Webmail UI                   |

These are only required when exposing services directly. With Cloudflare Tunnel they are handled by cloudflared and do not need to be open on your firewall.

### Environment Files

- `[.env.bulwark](./.env.bulwark)` -> Bulwark configuration (JMAP server URL, branding, secrets).
- `[.env.tunnel](./.env.tunnel)` -> Cloudflare tunnel token (keep this private, it's gitignored).

A `.env.bulwark.example` and `.env.tunnel.example` are provided as templates.
