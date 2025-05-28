# 🏠 Home Assistant on Raspberry Pi with Docker & NGINX Proxy Manager

This guide runs [Home Assistant](https://www.home-assistant.io/) in Docker on a Raspberry Pi and exposes it securely using NGINX Proxy Manager (NPM) with a custom domain and HTTPS.

---

## 🧰 Requirements

* Raspberry Pi (tested on Raspberry Pi 4)
* Docker & Docker Compose installed
* NGINX Proxy Manager (can be on same/different host)
* A domain (e.g., `home.example.com`) pointing to your public IP
* Port forwarding (80 and 443) to the NGINX Proxy Manager server

---

## ⚙️ Step 1: Deploy Home Assistant with Docker

Create the Docker Compose file on your Raspberry Pi:

### `docker-compose.yml`

```yaml
version: '3.7'

services:
  homeassistant:
    container_name: homeassistant
    image: ghcr.io/home-assistant/home-assistant:stable
    volumes:
      - /home/strider/homeassistant/config:/config
      - /etc/localtime:/etc/localtime:ro
      - /run/dbus:/run/dbus:ro
    restart: unless-stopped
    privileged: true
    network_mode: host
```

**Explanation:**

* `- /home/strider/homeassistant/config:/config` mounts the configuration directory from the host into the container. This allows Home Assistant to persist settings and configurations across restarts and upgrades. This is especially important when exposing Home Assistant via a **public domain**, as it allows for persistent SSL settings, user data, and UI preferences.

Launch the container:

```bash
docker-compose up -d
```

Access Home Assistant locally:

```
http://<raspberry-pi-ip>:8123
```

---

## 🌐 Step 2: Set Up Reverse Proxy via NGINX Proxy Manager

1. Log into NGINX Proxy Manager.
2. Go to **Proxy Hosts** > **Add Proxy Host**.

**Details:**

* **Domain Names**: `home.example.com`
* **Forward Hostname/IP**: `<raspberry-pi-ip>`
* **Forward Port**: `8123`
* **Enable**:

  * Websockets Support
  * Block Common Exploits

**SSL:**

* Check **Enable SSL**
* Select **Request a new SSL Certificate via Let’s Encrypt**
* Check **Force SSL**

**Advanced Tab** – Add the following:

```nginx
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;

proxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
```

**Why these settings are required:**

* `proxy_set_header Host $host;` ensures that the correct hostname is passed to Home Assistant, necessary for internal routing.
* `proxy_set_header X-Real-IP $remote_addr;` and `X-Forwarded-For` pass the original client's IP address to Home Assistant, useful for logging and security.
* `proxy_set_header X-Forwarded-Proto $scheme;` tells Home Assistant whether the original request was HTTP or HTTPS.
* `proxy_http_version 1.1;` is required for proper WebSocket support.
* `proxy_set_header Upgrade $http_upgrade;` and `Connection "upgrade";` explicitly enable WebSocket connections, which Home Assistant uses for its UI and real-time updates.

---

## 🔧 Step 3: Configure Home Assistant to Trust Proxy

Edit your `configuration.yaml` in Home Assistant config directory:

### `/home/strider/homeassistant/config/configuration.yaml`

```yaml
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 192.168.1.116  # Your NGINX Proxy Manager IP
    - 127.0.0.1      # Always safe to include localhost

mobile_app:
```

**Explanation:**

* `use_x_forwarded_for: true` allows Home Assistant to identify the real client IP when behind a proxy.
* `trusted_proxies:` lists the IPs that are allowed to send proxy headers. You must include your NGINX Proxy Manager's IP here, or Home Assistant will reject requests with `X-Forwarded-*` headers.
* `mobile_app:` is required to support features from the official Home Assistant mobile apps, such as notifications, location tracking, and sensor reporting.
* This `configuration.yaml` file is not a default file automatically generated—it is created as part of this setup in the `/config` volume to support customization when using a public domain.

Restart Home Assistant:

```bash
docker-compose down
docker-compose up -d
```

---

✅ **Done!**
You can now access Home Assistant securely via `https://home.example.com`.

Let me know if you want to automate this with Docker volumes, backups, or dynamic DNS.
