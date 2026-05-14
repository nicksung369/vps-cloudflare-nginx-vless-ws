---
name: vps-cloudflare-nginx-vless-ws
description: Use when deploying, verifying, or auditing a Cloudflare proxied VLESS WebSocket node on an Ubuntu VPS, especially Cloudflare DNS, Origin CA, Full strict TLS, nginx WebSocket reverse proxy, local-only Xray inbound, random WS paths, mihomo YAML, UFW, and origin exposure checks.
---

# Cloudflare nginx VLESS-WS Deploy

Deploy an Xray `VLESS + WebSocket` inbound behind nginx and a Cloudflare proxied DNS record. Use this when you have a domain and want clients to connect to the domain instead of the raw VPS IP. Use only where self-hosted proxy services are lawful and allowed by the VPS provider.

## Inputs

| Variable | Example | Notes |
|---|---|---|
| `DOMAIN` | `node.example.com` | Proxied Cloudflare DNS record |
| `VPS_IP` | `203.0.113.10` | Origin VPS IP used only in Cloudflare DNS |
| `NODE_NAME` | `cf-vless-ws` | Client display name; avoid spaces |
| `XRAY_PORT` | auto-generated | Localhost-only Xray port |
| `WS_PATH` | auto-generated | Random path such as `/a1b2...` |
| `UUID` | auto-generated | VLESS client UUID |

## Safety Rules

- Use a domain you control and a Cloudflare proxied DNS record.
- Put Xray on `127.0.0.1` only. Do not expose `XRAY_PORT` in UFW or cloud firewalls.
- Use a random `WS_PATH`; avoid obvious paths such as `/ws`, `/api`, or `/proxy`.
- Treat the UUID and WebSocket path as client credentials. Do not commit generated YAML or links.
- Prefer Cloudflare Origin CA plus **Full (strict)** mode for the origin TLS connection.

## 0) Cloudflare Preflight

In Cloudflare:

1. Add an `A` record for `DOMAIN` pointing to `VPS_IP`.
2. Set Proxy status to **Proxied**.
3. Create an Origin CA certificate for `DOMAIN` or `*.example.com`.
4. Set SSL/TLS encryption mode to **Full (strict)** after the certificate is installed on the VPS.

On the VPS:

```bash
set -euo pipefail

export DOMAIN="node.example.com"
export VPS_IP="203.0.113.10"
export NODE_NAME="${NODE_NAME:-cf-vless-ws}"

test "$(id -u)" -eq 0 || { echo "Run sudo -i first"; exit 1; }
test -n "$DOMAIN" && test -n "$VPS_IP"
```

## 1) Install Xray, nginx, and Tools

```bash
apt update
apt -y install nginx curl ca-certificates openssl jq
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install
command -v xray
xray version
nginx -v
```

## 2) Generate Local Port, Path, and UUID

```bash
choose_local_port() {
  local port
  while :; do
    port="$(shuf -i 20000-65000 -n 1)"
    if ! ss -tuln | awk '{print $5}' | grep -Eq "[:.]$port$"; then
      printf '%s\n' "$port"
      return 0
    fi
  done
}

export XRAY_PORT="${XRAY_PORT:-$(choose_local_port)}"
export WS_PATH="${WS_PATH:-/$(openssl rand -hex 16)}"
export UUID="${UUID:-$(xray uuid)}"

install -d -m 700 /root/vless-ws
cat > /root/vless-ws/secrets.env <<EOF
DOMAIN=$DOMAIN
VPS_IP=$VPS_IP
NODE_NAME=$NODE_NAME
XRAY_PORT=$XRAY_PORT
WS_PATH=$WS_PATH
UUID=$UUID
EOF
chmod 600 /root/vless-ws/secrets.env
```

## 3) Install Cloudflare Origin Certificate

Paste the Origin CA certificate and private key from Cloudflare:

```bash
install -d -m 700 /etc/nginx/cf

# Paste the full Origin CA certificate PEM into this file.
${EDITOR:-nano} /etc/nginx/cf/cert.pem

# Paste the full Origin CA private key PEM into this file.
${EDITOR:-nano} /etc/nginx/cf/key.pem

chmod 600 /etc/nginx/cf/cert.pem /etc/nginx/cf/key.pem
openssl x509 -in /etc/nginx/cf/cert.pem -noout -subject -issuer -dates
```

Do not commit the certificate key. If the key leaks, revoke and recreate the Origin CA certificate.

## 4) Configure Xray Local VLESS-WS Inbound

```bash
. /root/vless-ws/secrets.env

cat > /usr/local/etc/xray/config.json <<EOF
{
  "log": { "loglevel": "warning" },
  "inbounds": [
    {
      "tag": "vless-ws-local",
      "listen": "127.0.0.1",
      "port": $XRAY_PORT,
      "protocol": "vless",
      "settings": {
        "clients": [{ "id": "$UUID" }],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "ws",
        "wsSettings": { "path": "$WS_PATH" }
      },
      "sniffing": {
        "enabled": true,
        "destOverride": ["http", "tls"]
      }
    }
  ],
  "outbounds": [
    { "protocol": "freedom", "tag": "direct" }
  ]
}
EOF

xray run -test -config /usr/local/etc/xray/config.json
systemctl enable --now xray
systemctl restart xray
ss -tlnp | grep "127.0.0.1:$XRAY_PORT"
```

## 5) Configure nginx HTTPS and WebSocket Reverse Proxy

```bash
. /root/vless-ws/secrets.env

cat > /var/www/html/index.html <<'EOF'
<!doctype html>
<html lang="en">
<head><meta charset="utf-8"><title>OK</title></head>
<body><h1>OK</h1></body>
</html>
EOF

cat > "/etc/nginx/sites-available/$DOMAIN" <<EOF
server {
    listen 80;
    server_name $DOMAIN;
    return 301 https://\$host\$request_uri;
}

server {
    listen 443 ssl http2;
    server_name $DOMAIN;

    ssl_certificate     /etc/nginx/cf/cert.pem;
    ssl_certificate_key /etc/nginx/cf/key.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    root /var/www/html;
    index index.html;

    location = $WS_PATH {
        if (\$http_upgrade != "websocket") { return 404; }
        proxy_pass http://127.0.0.1:$XRAY_PORT;
        proxy_http_version 1.1;
        proxy_set_header Upgrade \$http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
        proxy_read_timeout 300s;
    }
}
EOF

ln -sf "/etc/nginx/sites-available/$DOMAIN" "/etc/nginx/sites-enabled/$DOMAIN"
rm -f /etc/nginx/sites-enabled/default
nginx -t
systemctl enable --now nginx
systemctl reload nginx
```

## 6) Firewall

```bash
ufw allow 80/tcp comment "HTTP redirect"
ufw allow 443/tcp comment "HTTPS via Cloudflare"
ufw status verbose

# XRAY_PORT must not be public.
ufw status verbose | grep "$XRAY_PORT" && { echo "Do not expose XRAY_PORT"; exit 1; } || true
ss -tlnp | grep "127.0.0.1:$XRAY_PORT"
```

## 7) Generate mihomo YAML

```bash
. /root/vless-ws/secrets.env

cat > "/root/vless-ws/${NODE_NAME}.yaml" <<EOF
proxies:
  - name: ${NODE_NAME}
    type: vless
    server: ${DOMAIN}
    port: 443
    uuid: ${UUID}
    network: ws
    tls: true
    udp: true
    ws-opts:
      path: ${WS_PATH}
      headers:
        Host: ${DOMAIN}
    servername: ${DOMAIN}
EOF

chmod 600 "/root/vless-ws/${NODE_NAME}.yaml"
```

## Verification

```bash
dig +short "$DOMAIN" 2>/dev/null || true
curl -I "https://$DOMAIN"
curl -I "https://$DOMAIN$WS_PATH" || true
xray run -test -config /usr/local/etc/xray/config.json
systemctl is-active xray
systemctl is-active nginx
nginx -t
ss -tlnp | grep "127.0.0.1:$XRAY_PORT"
journalctl -u xray -n 80 --no-pager
```

After importing the YAML into a client, test from the client machine:

```bash
curl --proxy socks5://127.0.0.1:7891 https://ifconfig.me
```

The visible IP may be a Cloudflare edge IP from the destination's perspective, while Xray egress traffic exits from the VPS.

## Gotchas Worth Memorizing

- Cloudflare Origin CA certificates are compatible with Full (strict) mode, but browsers do not trust them directly if the record is unproxied.
- `XRAY_PORT` must bind to `127.0.0.1` and must not be opened in UFW.
- Use a random `WS_PATH` of at least 16 bytes of hex. Obvious paths get scanned.
- The client `server` and `servername` are the domain, not the VPS IP.
- Cloudflare proxying hides the origin from casual clients, but origin IP can still leak through old DNS records, direct services, logs, or provider panels.
- If HTTPS returns Cloudflare `526`, the Origin CA certificate does not match the hostname, is expired, or Full (strict) was enabled before nginx presented the cert.
