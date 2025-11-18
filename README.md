
## Vaultwarden on Ugreen NAS (UGOS): LAN-Only Setup (Let's Encrypt TLS)

This guide details the installation of Vaultwarden on an Ugreen NAS (UGOS) using Docker Compose, Caddy as a reverse proxy, and **Cloudflare for Let's Encrypt TLS certificate management**.

**Objective:** To secure Vaultwarden with a valid, publicly trusted SSL/TLS certificate (**Let's Encrypt**) while restricting actual access to the **Local Area Network (LAN) only** eg. https://vault.domain:8443

> **Why this configuration?**  *Some users prefer to **access their Vaultwarden instance exclusively locally** and prefer not to expose it to the public internet.* 
> 
> **Then why not just use the NAS's IP address directly?**  *Vaultwarden requires the use of **HTTPS (SSL/TLS)**. While self-signed certificates are accepted, they often cause **trust errors and connection issues** in dedicated client applications such as those for iOS, Android, and desktop, due to strict SSL/TLS validation policies.*

****NOTE:** Ports **80 and 443 are reserved by UGOS** and **cannot be used**, regardless of custom UGOS port settings.** 

## 0. Prerequisites 

   - **Host System:** Ugreen NAS running **UGOS**.
   - **Domain Management:** An active **domain name** with **Cloudflare DNS** configured. [Cloudflare](https://www.cloudflare.com/)
   - **Containerization:** **Docker** and **Portainer** installed and running on the NAS. [Docker & Portainer Documentation for UGOS](https://mariushosting.com/how-to-install-portainer-on-your-ugreen-nas/) 

## 1. Cloudflare DNS Setup and API Token Creation

 1. **Point DNS to Your Server** 
	 - **Configure a Custom DNS Record**: In your Cloudflare DNS settings, create an A record (or CNAME if preferred) for your Vaultwarden service (e.g., vault.domain.com). 
	 - **Crucial for Local Access**: Set this record to point your server IP address (e.g., 192.168.1.50) and ensure the Cloudflare Proxy (orange cloud) is DISABLED (DNS Only mode). This is necessary to satisfy Let's Encrypt validation. 

2. **Cloudflare API Token Creation**
	You need an API Token so that **Caddy** can communicate with Cloudflare to perform the **DNS-01 challenge** required by Let's Encrypt. This allows Caddy to obtain an SSL certificate for your domain without externally.

	-   **Access API Tokens:** Navigate to the "API Tokens" section in your Cloudflare dashboard.
	-   **Create a Custom Token:** Create a new token with the following permissions, which are the minimum required for the DNS-01 challenge:
    -   **Permissions:** Zone -> DNS -> Edit
    -   **Zone Resources:** `Include -> Specific Zone -> [Your Domain Name]`
	-   **Token Retrieval:** Copy the generated token immediately. **This token will be used in the stack configuration as an environment variable (`CLOUDFLARE_API_TOKEN`).**


## 2. Building a Custom Caddy Image with the Cloudflare Module

- Since the standard Caddy image does not include the DNS challenge module for Cloudflare by default, we must **build a custom image** that incorporates it. This is a requirement for obtaining Let's Encrypt certificates when ports 80/443 are reserved by UGOS.

	- Navigate to **Portainer**.
	- Go to the **Images** section in the sidebar.
	-  Click on **Build a new image** (or similar option depending on your Portainer version).
	- **Name the Image:** Enter `caddy-custom:latest` for the tag.
	- In the web editor, paste the code bellow & **click build image**
```bash
# Use the official Caddy Builder image to compile the custom binary

FROM caddy:builder AS builder
RUN xcaddy build \
	--with github.com/caddy-dns/cloudflare
FROM caddy:latest
COPY --from=builder /usr/bin/caddy /usr/bin/caddy
```

## 3. Creating the Folder Structure and Caddyfile

This step involves setting up the necessary directory structure on your NAS to hold the persistent data and configuration files for your Docker containers.
```bash
# Location: In a shared volume, e.g., /volumeX/shared_folder/docker/
# This path is accessible via the UGOS File explorer.

docker/
	├── caddy/
	│   ├── data/
	│   ├── config/
	│   └── Caddyfile # Caddy configuration file
	└── vaultwarden
	    └── data/
```

### Updating the Caddyfile
You must update the `Caddyfile` with your specific domain name and the alternative port (`8443` in this case) to respect the UGOS port restriction. Proxy all incoming traffic to the Vaultwarden container, which is running internally on the standard HTTP port **80** (accessible via the service name `vaultwarden` within the Docker network).

**Action:** Update the `Caddyfile` below, ensuring you **change the domain name only** to match your configuration.
```bash
vault.domain.com:8443 {
    tls {
        dns cloudflare {env.CLOUDFLARE_API_TOKEN}
    }
    reverse_proxy vaultwarden:80
}
```


## 4. Deploying the Vaultwarden Stack

This step defines and launches both the Vaultwarden and Caddy services together, ensuring they communicate correctly and manage the persistent data.
### Instructions in Portainer
1.  In Portainer, create a new stack (e.g., `vaultwarden`).
2.  In the web editor, place the code below, making sure to define your required **Environment Variables** (see note below).
#### ⚠️ Environment Variables Setup (Crucial)
-  `ADMIN_TOKEN`: **OPTIONAL FOR TESTING** (e.g., `testing-admin-token`). **For production use, you MUST use an Argon2 hash** (see [Vaultwarden Admin Page Docs](https://github.com/dani-garcia/vaultwarden/wiki/Enabling-admin-page#using-argon2)).
-  `DOMAIN_NAME`: Your full domain (e.g., `vault.domain.com`).
-   `CLOUDFLARE_API_TOKEN`: **MANDATORY**, obtained in Step 1.

```yml 
version: '3.8'

services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped
    environment:
      - ADMIN_TOKEN=testing-admin-token
      - WEBSOCKET_ENABLED=true
      - DOMAIN=https://vault.domain.com:8443
    volumes:
      - /volume1/docker/vaultwarden/data:/data
    networks:
      - internal
  caddy:
    image: caddy-custom:latest
    container_name: caddy
    restart: unless-stopped
    environment:
      - CLOUDFLARE_API_TOKEN=-abcdefg123456789ABCDEFG
    volumes:
      - /volume1/docker/caddy/data:/data
      - /volume1/docker/caddy/config:/config
      - /volume1/docker/caddy/Caddyfile:/etc/caddy/Caddyfile:ro
    ports:
      - "8443:8443"
    networks:
      - internal
networks:
  internal:
    driver: bridge
```

- Click **Deploy the stack**.
    

## 5. Final Access

Once the stack is successfully deployed and Caddy has retrieved the certificate (check Caddy logs), you should be able to access the interface and connect your clients.

**Result:** You should now be able to access the interface via `https://vault.domain.com:8443` and connect to your clients (app/desktop/extension) using the same URL.

-----

#### External Access (Optional)
If you wish to access your Vaultwarden instance externally, look into [Cloudflare Tunnels](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/). This is the most secure method, as it does not require opening any ports on your router.

*You are **solely responsible** for the security of your NAS, the configuration of your network, and the protection of your secrets (including the `CLOUDFLARE_API_TOKEN` and the Vaultwarden `ADMIN_TOKEN`). **Proceed at your own risk.***