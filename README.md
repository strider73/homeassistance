 🏠 Home Assistant on Raspberry Pi with Docker & NGINX Proxy Manager

This setup runs [Home Assistant](https://www.home-assistant.io/) in Docker on a Raspberry Pi and securely exposes it using NGINX Proxy Manager with a custom domain.

---

## 🧰 Requirements

- Raspberry Pi (tested on Raspberry Pi 4)
- Docker & Docker Compose installed
- NGINX Proxy Manager set up (can run on the same or a different server)
- A domain (e.g., `home.example.com`) pointing to your public IP
- Port forwarding (ports 80 and 443) to NGINX Proxy Manager

---

## ⚙️ Step 1: Deploy Home Assistant with Docker

Create the Docker Compose file:

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
Launch the container
bash
Copy
Edit
docker-compose up -d
Access Home Assistant locally at:

cpp
Copy
Edit
http://<raspberry-pi-ip>:8123
🌐 Step 2: Set Up Reverse Proxy via NGINX Proxy Manager
Log into NGINX Proxy Manager.

Add a Proxy Host:

Domain Name: home.example.com

Forward Hostname / IP: <raspberry-pi-ip>

Forward Port: 8123

Enable: Websockets Support

Enable: Block Common Exploits

Under SSL:

Check: Enable SSL

Choose: Request a new SSL certificate via Let’s Encrypt

Enable: Force SSL

Advanced Configuration (Important!)
Add this under the “Advanced” tab:

nginx
Copy
Edit
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;

proxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
🔧 Step 3: Configure Home Assistant to Trust Proxy
Edit /home/strider/homeassistant/config/configuration.yaml:

yaml
Copy
Edit
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 192.168.1.116  # IP address of your NGINX Proxy Manager host
    - 127.0.0.1
Restart Home Assistant
bash
Copy
Edit
docker-compose down
docker-compose up -d