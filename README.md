# 🛡️ Pi-hole + Unbound (Docker Edition) - Power User

![intropihole.png](assets/intropihole.png)

> "You are the master of your own network."

This project deploys a professional **Ad Blocking** and **DNS Sovereignty** solution using **Docker** containers.
It replaces your ISP's or Google's DNS with a private, encrypted infrastructure that is 100% controlled by you.

---

### 🛑 STOP: READ BEFORE USE

**This project is NOT "Plug & Play".**
For this to work on your home network, **you must** edit the configuration files with your own data. If you skip this, it will not work.

**Mandatory Changes:**

1.  **Your Local IP:** The code assumes an example IP. You must configure your Router to assign a **Static IP** to your server and use that IP to access the dashboard.
2.  **Your Password:** The file comes with an example password (`change_me_please`) which you must change for security.

---

### Software Architecture

* **[Pi-hole](https://pi-hole.net/):** The Shield ("DNS Sinkhole"). It intercepts and blocks requests for ads, trackers, and telemetry at the network level.
* **[Unbound](https://github.com/klutchell/unbound):** The Recursive Resolver. It queries the Internet Root Servers directly, ensuring data integrity and removing intermediaries (ISP, Google, Cloudflare).
* **[Docker](https://www.docker.com/):** Ensures an isolated, reproducible, and clean deployment.

---

## Privacy & Sovereignty Philosophy

As I am specializing in data sovereignty and privacy methods that users can implement, this guide allows you to easily become the master of your own network.

This project strictly adheres to the **Data Sovereignty** principle:

* **Zero Reliance:** We do not rely on upstream providers like Google (`8.8.8.8`) or Cloudflare (`1.1.1.1`).
* **Privacy by Design:** By using recursive resolution, no single entity centralizes your browsing history. You are your own DNS provider.
* **Security:** DNSSEC validation implemented in Unbound to prevent DNS spoofing.

---

## Installation & Deployment

### 1. Prerequisites

* **Hardware:** Raspberry Pi 4 (4GB RAM) or higher (Recommended configuration).
* **Software:** Docker and Docker Compose installed.
* **Network:** Port **53** must be free on the host machine and a **Static IP** assigned to the server.

### 2. File Structure

Clone this repository or create the following folder structure.
*Note: The `.gitignore` file is vital to avoid uploading logs and private data to GitHub.*

```text
privacy-shield/
├── docker-compose.yml
├── .gitignore
└── unbound/
    └── unbound.conf
```

**Recommended content for `.gitignore`:**

```text
config/pihole/
etc-pihole/
etc-dnsmasq.d/
*.log
.DS_Store
.env
```

### 3. Unbound Configuration (`unbound/unbound.conf`)

**Optimized for Raspberry Pi 4/5 (4GB+ RAM) using the `klutchell/unbound` image.**

```yaml
server:
    # 1. Network Config
    verbosity: 0
    interface: 0.0.0.0
    port: 53
    do-ip4: yes
    do-udp: yes
    do-tcp: yes
    
    # 2. Access Control (Allow private ranges)
    access-control: 10.0.0.0/8 allow
    access-control: 172.16.0.0/12 allow
    access-control: 192.168.0.0/16 allow

    # 3. Privacy & Security
    hide-identity: yes
    hide-version: yes
    harden-glue: yes
    harden-dnssec-stripped: yes
    use-caps-for-id: no

    # 4. Hardware Optimization (RPi 4 4GB+)
    edns-buffer-size: 1232
    prefetch: yes
    num-threads: 2
    so-rcvbuf: 4m
    so-sndbuf: 4m
    msg-cache-slabs: 4
    rrset-cache-slabs: 4
    infra-cache-slabs: 4
    key-cache-slabs: 4
    msg-cache-size: 50m
    rrset-cache-size: 100m
    cache-min-ttl: 3600
    cache-max-ttl: 86400

    # 5. Specific paths for klutchell/unbound image
    root-hints: "/etc/unbound/root.hints"
    auto-trust-anchor-file: "/var/run/unbound/root.key"
```

### 4. Service Definition (`docker-compose.yml`)

We use a static internal network (`10.10.10.0/24`) and map the web panel to port **8080**.

```yaml
services:
  unbound:
    container_name: unbound
    image: klutchell/unbound:latest
    restart: unless-stopped
    hostname: unbound
    volumes:
      # Correct mapping for klutchell image (/etc/unbound)
      - ./unbound/unbound.conf:/etc/unbound/unbound.conf:ro
    networks:
      privacy_net:
        ipv4_address: 10.10.10.2

  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    restart: unless-stopped
    hostname: pihole
    depends_on:
      - unbound
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      # Mapping container port 80 to host port 8080
      # to avoid conflicts and access via :8080
      - "8080:80/tcp" 
    environment:
      - TZ=Europe/Madrid
      # ⚠️ SECURITY NOTICE: Change this password immediately after deployment
      - WEBPASSWORD=change_me_please
      - PIHOLE_DNS_=10.10.10.2#53
      - DNSSEC=true
      - DNSMASQ_LISTENING=all
    volumes:
      - ./etc-pihole:/etc/pihole
      - ./etc-dnsmasq.d:/etc/dnsmasq.d
    networks:
      privacy_net:
        ipv4_address: 10.10.10.3

networks:
  privacy_net:
    driver: bridge
    ipam:
      config:
        - subnet: 10.10.10.0/24
```

### 5. Deployment

Run the following command in the project root:

```bash
docker compose up -d
```

---

## Dashboard Access

Once deployed, you must access the web panel to finish the configuration.
The web address is **NOT automatic**; it depends on the IP assigned to your server (Raspberry Pi/PC).

**URL Format:** `http://<YOUR_LOCAL_IP>:8080/admin`

**Examples:**

* If your Raspberry IP is `192.168.1.100`, go to:
    👉 **`http://192.168.1.100:8080/admin`**
* If your Raspberry IP is `192.168.0.50`, go to:
    👉 **`http://192.168.0.50:8080/admin`**

*(Note: We use port `:8080` because we defined it in the `docker-compose.yml` file).*

![image.png](assets/pihole-main.png)

---

## ⚙️ Critical Post-Install Configuration

Mandatory steps in the web dashboard:

### 1. Link to Unbound (Upstream DNS)

* Go to **Settings > DNS**.
* **Uncheck** all external providers (Google, OpenDNS, etc).
* In **"Custom 1 (IPv4)"**, type: 👉 **`10.10.10.2#53`**
* **IMPORTANT:** Ensure the **"Use DNSSEC"** box is **CHECKED**.
  * *Why?* Unbound performs the validation, but Pi-hole needs this option enabled to pass the "Authenticated Data" (ad) flag to your devices, confirming the security chain is intact.
---
![image.png](assets/image%201.png)

### 2. Permit Docker Traffic (CRITICAL) ⚠️

* In the same DNS tab, under **"Interface Settings"**.
* Select the option: 👉 **"Permit all origins"**.

> **Why choose the "Dangerous" option?**
> Pi-hole labels this option as dangerous for cloud-exposed servers.
> However, **in Docker, it is MANDATORY**.
> Due to how Docker handles networking (NAT), requests from your devices appear to come from an "external" network (the internal Docker network 10.10.10.x) instead of your local network. If you choose "Allow only local requests", Pi-hole will block your own devices.
>
> **🛡️ Security Note:** This setting is **100% safe** for home use as long as you **DO NOT open port 53** on your router to the internet.

### 3. Add Blocklists (Adlists)

1.  Go to **Group Management > Adlists**.
2.  Add these recommended URLs:
    * **StevenBlack Unified:** `https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts`
    * **AdGuard DNS Filter:** `https://raw.githubusercontent.com/AdguardTeam/FiltersRegistry/master/filters/filter_15_DnsFilter/filter.txt`
    * **Firebog Green:** `https://v.firebog.net/hosts/AdguardDNS.txt`

![image.png](assets/image%202.png)

3.  Go to **Tools > Update Gravity** and click **Update**.

![image.png](assets/image%203.png)

---

## Router Configuration

⚠️ **ATTENTION:** Perform these steps **ONLY** after you have completed the above steps and verified that you can access the Pi-hole dashboard.

### 1. IP Reservation (DHCP Reservation)

This ensures your server always has the same address (e.g., `192.168.1.100`).

* Log in to your router -> **LAN/DHCP**.
* Look for **"Static Lease"**, **"Address Reservation"**, or **"Bind IP to MAC"**.
* Assign a static IP to your Raspberry Pi's MAC address.

### 2. The "Litmus Test" (Safety Check)

Before changing the router, test on your own PC:

1.  Manually change your Windows/Linux network card DNS to your Pi-hole IP (`192.168.1.100`).
2.  Try to browse. Does it work? Are ads blocked?
    * **YES:** Proceed to the next step.
    * **NO:** Check your configuration. **DO NOT TOUCH THE ROUTER YET.**

### 3. Change Router DNS Server

If the test was successful, make the global change:

* In the router's **DHCP** settings:
    * **Primary DNS:** Enter your Raspberry Pi's static IP (e.g., `192.168.1.100`).
    * **Secondary DNS:** **LEAVE EMPTY** or repeat the same IP.
    * ⚠️ **Error:** Do NOT put `8.8.8.8` as secondary, or you will bypass ad blocking.

### 4. The IPv6 Leak ⚠️

If your router assigns DNS via IPv6, devices will bypass Pi-hole.
**Solution:** Disable **DHCPv6** on the router or disable IPv6 on your devices.

---

## Verification (Freedom Test)

1.  Visit [DNSLeakTest.com](https://www.dnsleaktest.com/).
2.  Run the **Extended Test**.
3.  **Result:** You should see **ONLY your ISP's IP** (Digi, Movistar, Verizon, etc.). If you see Google or Cloudflare, check your router settings.

---

## Troubleshooting

### Port 53 Conflict (Ubuntu/Linux)

If `systemd-resolved` occupies port 53 on the host:

1.  Edit: `sudo nano /etc/systemd/resolved.conf` -> set `DNSStubListener=no`.
2.  Restart: `sudo systemctl restart systemd-resolved`.
3.  Link: `sudo ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf`.

### Password Reset (Pi-hole Web Access)

If you want to change the password after installation:

1.  **Enter container:** `docker exec -it pihole /bin/bash`
2.  **Execute:** `pihole setpassword your_password`
3.  **Exit:** `exit`

---

## 🙌 Acknowledgements

This project is possible thanks to the Open Source code of:

* [**Pi-hole Team**](https://pi-hole.net/)
* [**Unbound (Klutchell Docker Image)**](https://github.com/klutchell/unbound)
* [**Docker**](https://www.docker.com/)
* [**StevenBlack Hosts**](https://github.com/StevenBlack/hosts)

---

## 📜 License

MIT Licence

---

### 👤 Author

**[José Álvarez Dominguez]**

* [GitHub Profile](https://github.com/tu-usuario)
* [LinkedIn](https://www.linkedin.com/in/jadomin/)
