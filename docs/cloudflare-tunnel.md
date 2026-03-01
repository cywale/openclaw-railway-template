# Cloudflare Tunnel Setup for OpenClaw

This guide explains how to expose your OpenClaw deployment to the internet securely using Cloudflare Tunnels instead of Railway's public networking.

## Why Use Cloudflare Tunnels?

- **Enhanced Security**: No need to expose ports directly to the internet
- **DDoS Protection**: Cloudflare's network provides automatic DDoS mitigation
- **Custom Domains**: Use your own domain with SSL/TLS included
- **Traffic Analytics**: Get insights into your bot's traffic patterns
- **Access Control**: Add Cloudflare Access for additional authentication layers

## Prerequisites

1. A Cloudflare account (free tier works)
2. A domain managed by Cloudflare
3. Your OpenClaw deployed on Railway (or running locally)

## Setup Methods

### Option 1: Railway Deployment with Cloudflare Tunnel

#### Step 1: Create a Cloudflare Tunnel

1. Log in to the [Cloudflare Dashboard](https://dash.cloudflare.com/)
2. Go to **Zero Trust** → **Networks** → **Tunnels**
3. Click **Create a tunnel**
4. Choose **Cloudflared** as the tunnel type
5. Name your tunnel (e.g., `openclaw-production`)
6. Click **Save tunnel**

#### Step 2: Get Tunnel Credentials

After creating the tunnel, Cloudflare will provide you with a **tunnel token**. This is a long string that looks like:
```
eyJhIjoiY2xvdWRmbGFyZS10d...
```

**Important**: Save this token securely - you'll need it for Railway configuration.

#### Step 3: Configure Public Hostname

1. In the tunnel configuration, go to the **Public Hostnames** tab
2. Click **Add a public hostname**
3. Configure:
   - **Subdomain**: Choose a subdomain (e.g., `openclaw`)
   - **Domain**: Select your domain
   - **Type**: HTTP
   - **URL**: `localhost:8080` (or your Railway internal URL)
4. Click **Save hostname**

Your OpenClaw will be accessible at `https://openclaw.yourdomain.com`

#### Step 4: Add Tunnel to Railway (Sidecar Method)

Railway doesn't natively support running multiple processes in one container easily, so we'll use a startup script approach.

1. Create a new file `start-with-tunnel.sh` in your repo:

```bash
#!/bin/bash
set -e

# Start cloudflared in the background
cloudflared tunnel --no-autoupdate run --token $CLOUDFLARE_TUNNEL_TOKEN &

# Start the main application
exec node src/server.js
```

2. Make it executable:
```bash
chmod +x start-with-tunnel.sh
```

3. Update your `Dockerfile` to install cloudflared:

Add after line 51 (in the runtime image section):
```dockerfile
# Install cloudflared for Cloudflare Tunnel
RUN curl -L --output cloudflared.deb https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb \
  && dpkg -i cloudflared.deb \
  && rm cloudflared.deb
```

4. Update the CMD in `Dockerfile`:
```dockerfile
COPY start-with-tunnel.sh ./
RUN chmod +x start-with-tunnel.sh
CMD ["./start-with-tunnel.sh"]
```

5. Add the tunnel token to Railway:
   - Go to your Railway project
   - Add a new variable: `CLOUDFLARE_TUNNEL_TOKEN`
   - Paste your tunnel token as the value
   - Also add: `USE_CLOUDFLARE_TUNNEL=true` (optional, for conditional logic)

6. Redeploy your Railway project

#### Step 5: Configure OpenClaw to Use Internal Port Only

Since Cloudflare Tunnel will handle external traffic, you can optionally:

1. In Railway, **disable** Public Networking if you only want access via Cloudflare
2. The app will still listen on port 8080 internally
3. Cloudflare Tunnel will forward traffic from your domain to localhost:8080

---

### Option 2: Docker Compose with Cloudflare Tunnel (Local Development)

For local testing, use this Docker Compose configuration:

```yaml
version: '3.8'

services:
  openclaw:
    build: .
    ports:
      - "8080:8080"
    environment:
      - PORT=8080
      - SETUP_PASSWORD=test
      - OPENCLAW_STATE_DIR=/data/.openclaw
      - OPENCLAW_WORKSPACE_DIR=/data/workspace
    volumes:
      - openclaw-data:/data
    networks:
      - openclaw-network

  cloudflare-tunnel:
    image: cloudflare/cloudflared:latest
    command: tunnel --no-autoupdate run --token ${CLOUDFLARE_TUNNEL_TOKEN}
    environment:
      - CLOUDFLARE_TUNNEL_TOKEN=${CLOUDFLARE_TUNNEL_TOKEN}
    depends_on:
      - openclaw
    networks:
      - openclaw-network
    restart: unless-stopped

volumes:
  openclaw-data:

networks:
  openclaw-network:
    driver: bridge
```

To use:

1. Create a `.env` file:
```bash
CLOUDFLARE_TUNNEL_TOKEN=your-tunnel-token-here
```

2. Configure the Cloudflare Tunnel public hostname to point to:
   - **URL**: `openclaw:8080` (Docker service name)

3. Start the stack:
```bash
docker-compose -f docker-compose.tunnel.yml up
```

---

### Option 3: Manual cloudflared Installation (Existing Deployment)

If you already have OpenClaw running, you can run cloudflared separately:

#### On your server/VM:

```bash
# Install cloudflared
curl -L --output cloudflared.deb \
  https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared.deb

# Run the tunnel
cloudflared tunnel --no-autoupdate run --token YOUR_TUNNEL_TOKEN
```

#### As a systemd service:

Create `/etc/systemd/system/cloudflared.service`:

```ini
[Unit]
Description=Cloudflare Tunnel
After=network.target

[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/cloudflared tunnel --no-autoupdate run --token YOUR_TUNNEL_TOKEN
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Enable and start:
```bash
sudo systemctl enable cloudflared
sudo systemctl start cloudflared
sudo systemctl status cloudflared
```

---

## Testing Your Setup

1. **Check Tunnel Status**:
   - In Cloudflare Dashboard → Zero Trust → Networks → Tunnels
   - Your tunnel should show as "Healthy" with a green indicator

2. **Access Your Bot**:
   - Visit `https://openclaw.yourdomain.com/setup`
   - You should see the OpenClaw setup page

3. **Verify SSL**:
   - Cloudflare automatically provides SSL/TLS
   - Check the padlock icon in your browser

---

## Troubleshooting

### Tunnel Shows as "Down"

- Check that cloudflared is running: `ps aux | grep cloudflared`
- Check Railway logs for errors
- Verify the tunnel token is correct

### "Bad Gateway" Error

- Ensure OpenClaw is running and listening on port 8080
- Check that the public hostname URL is correct (http://localhost:8080 or http://openclaw:8080)
- Verify railway networking is working: check Railway logs

### Can't Access Setup Page

- Verify the SETUP_PASSWORD environment variable is set
- Check that /setup is not blocked by Cloudflare Access rules
- Try accessing the health endpoint: `/setup/healthz`

---

## Security Considerations

### 1. Disable Railway Public Networking

Once Cloudflare Tunnel is working, you can disable Railway's public networking for an extra layer of security.

### 2. Add Cloudflare Access

For additional protection:

1. Go to **Zero Trust** → **Access** → **Applications**
2. Create a new application for your OpenClaw domain
3. Set authentication rules (e.g., require email verification)
4. Apply to specific paths like `/setup/*`

### 3. Setup Logging

Enable Cloudflare logs to monitor access:
- Go to **Analytics & Logs** → **Logs**
- Configure Logpush to send logs to your preferred destination

---

## Advanced: Multiple Environments

You can create separate tunnels for different environments:

```bash
# Production
openclaw.yourdomain.com → Railway Production

# Staging
openclaw-staging.yourdomain.com → Railway Staging

# Development
openclaw-dev.yourdomain.com → Local Docker
```

Each environment gets its own tunnel token and configuration.

---

## Cost Considerations

- **Cloudflare Tunnels**: Free on all Cloudflare plans
- **Cloudflare Access**: Free for up to 50 users, $3/user/month after
- **Railway**: Standard Railway pricing applies

---

## Additional Resources

- [Cloudflare Tunnel Documentation](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/)
- [Railway Documentation](https://docs.railway.app/)
- [cloudflared GitHub](https://github.com/cloudflare/cloudflared)
