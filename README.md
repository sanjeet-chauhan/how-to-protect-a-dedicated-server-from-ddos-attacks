# How to Protect a Dedicated Server from DDoS Attacks

It usually starts the same way. Your site is fine in the morning, then pages begin to hang, the server load shoots up, and your monitoring shows traffic that makes no business sense. By the time you open the logs, thousands of requests are hitting you every second from places you have never seen before.

That is a DDoS attack, and a dedicated server does not make you immune to it. What a dedicated server does give you is control. You get the whole machine, full root access and a clean network setup, so you can build real defences instead of hoping for the best.

This guide shows you exactly how to do that on a YouStable [dedicated server in India](https://www.youstable.com/dedicated-servers/india/), from the first day to the day an attack actually happens.

## Know What You Are Defending Against

DDoS attacks are not all the same, and each type breaks something different. Once you know which layer is under pressure, the fix becomes much clearer.

| Attack Type | What It Does | Typical Sign |
|---|---|---|
| Volumetric (UDP floods, DNS or NTP amplification) | Fills your network uplink with junk traffic | Site unreachable even though the server itself looks idle |
| Protocol (SYN floods) | Exhausts the server's connection table | Many half-open connections, new visitors cannot connect |
| Application layer (HTTP floods) | Sends heavy page requests that look real | High CPU and RAM, slow database, busy web server |

### Where Server Level Protection Stops

Be honest about this part, because it decides your whole plan. Your firewall, kernel settings and web server tuning can handle protocol and application attacks very well. But if a volumetric flood fills the uplink before it reaches your server, nothing running inside the server can stop it. That traffic has to be filtered upstream, on the network side.

This is why your uplink and your provider's network matter. Every YouStable dedicated server ships with a 1 Gbps uplink, and you can scale to 10 Gbps on request. A dedicated network operations team monitors your server around the clock and acts on alerts in real time. For large network level attacks, talk to support about what can be done for your specific server and traffic profile.

## Step by Step: Hardening Your Dedicated Server

Your YouStable dedicated server is semi-managed. That means the hardware, network and data center are looked after, while the operating system, firewall and application security are in your hands. The steps below cover exactly that part.

### Part 1: Start with the Right Foundation

**Step 1. Pick a plan with room to spare.** DDoS pressure shows up as CPU, RAM and connection load, so do not buy a server that is already running at its limit. Here are the stock plans on the [India dedicated server page](https://www.youstable.com/dedicated-servers/india/).

| Processor | Cores / Speed | RAM | Storage | Bandwidth | Price (12 months) |
|---|---|---|---|---|---|
| E3-1230v3 | 4 Core / 3.3 GHz | 32 GB DDR4 | 480 GB SSD | 10 TB | $77.99/mo |
| E3-123x v5/v6 | 4 Core / 3.2 GHz | 32 GB DDR4 | 480 GB SSD | 10 TB | $103.99/mo |
| E-2136 (Most Popular) | 6 Core / 3.3 GHz | 32 GB DDR4 | 480 GB SSD | 10 TB | $117.99/mo |
| E-2286G | 6 Core / 4 GHz | 32 GB DDR4 | 480 GB NVMe SSD | 10 TB | $143.99/mo |
| E5-2667v4 | 8 Core / 3.2 GHz | 32 GB DDR4 | 960 GB NVMe SSD | 10 TB | $169.99/mo |

The 12 month cycle saves 15% over the regular monthly price. If your site is a likely target, such as a store, a financial platform or a high traffic portal, choose a higher plan or request a custom build with up to 512 GB RAM and a 10 Gbps uplink.

**Step 2. Order and add the security tools.** Click *Configure Now* on your plan, choose your data center and operating system, and add these optional extras if you want them from day one.

| Add-on | Starting Price | Why It Helps |
|---|---|---|
| Imunify360 | ₹950 | Firewall, malware scanner and intrusion detection |
| LiteSpeed web server | ₹950 | Faster page loads and better concurrency under load |
| Premium Support | ₹1,792 | Priority response and proactive semi-managed monitoring |
| Additional IPv4 | ₹475 | Separate IPs for isolating services |

Sign up or log in on [YouStable](https://www.youstable.com/), pay in INR by UPI, card or net banking, and you will get a GST compliant invoice.

**Step 3. Choose the location nearest to your visitors.** Mumbai, Noida, Indore and Hyderabad are available. A closer location means lower latency for real users, which also makes it easier to spot traffic that does not belong.

### Part 2: Lock Down the Network

**Step 4. Close everything you do not use.** A server with fewer open ports has fewer things to attack. Allow only web traffic and your SSH access.

On Ubuntu or Debian with ufw:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw limit 22/tcp
sudo ufw enable
```

On AlmaLinux or Rocky Linux with firewalld:

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --remove-service=cockpit
sudo firewall-cmd --reload
```

Make sure SSH is allowed before you enable the firewall, or you will lock yourself out. Since you have full root access, you can always fix a mistake from the panel, but it is better not to need it.

**Step 5. Tune the kernel against SYN floods.** Create a file at `/etc/sysctl.d/99-ddos-protection.conf` with these lines:

```bash
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 4096
net.ipv4.tcp_synack_retries = 2
net.core.somaxconn = 4096
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.conf.all.accept_source_route = 0
```

Then apply it:

```bash
sudo sysctl --system
```

SYN cookies let the server keep answering real visitors while a SYN flood is running. Always test these values on your own workload, because ideal numbers depend on your traffic.

**Step 6. Limit connections per IP, carefully.** If one address opens hundreds of connections, that is rarely a real visitor. On setups that use iptables, you can cap it:

```bash
sudo iptables -A INPUT -p tcp --syn --dport 443 -m connlimit --connlimit-above 100 --connlimit-mask 32 -j REJECT --reject-with tcp-reset
```

One warning for Indian websites: many mobile users sit behind carrier networks where thousands of people share a small number of public IPs. If you set this limit too low, you will block genuine customers. Start generous and tighten slowly while watching your logs.

**Step 7. Secure SSH.** Attackers often combine a DDoS with brute force attempts. Use key based login, disable password login, and add fail2ban.

In `/etc/ssh/sshd_config`:

```bash
PermitRootLogin prohibit-password
PasswordAuthentication no
```

Restart SSH after testing your key login in a second window, and install fail2ban with a basic jail in `/etc/fail2ban/jail.local`:

```bash
[sshd]
enabled = true
maxretry = 5
findtime = 10m
bantime = 1h
```

```bash
sudo systemctl enable --now fail2ban
```

### Part 3: Protect the Web Layer

**Step 8. Rate limit requests at the web server.** Application layer floods look like normal visitors, so you slow them down instead of blocking them outright. On nginx, for example:

```nginx
limit_req_zone $binary_remote_addr zone=perip:10m rate=10r/s;

server {
    location / {
        limit_req zone=perip burst=20 nodelay;
    }
}
```

If you use LiteSpeed, it has its own per client connection and request throttling settings, so configure those instead. With a control panel such as cPanel, Plesk or DirectAdmin, check what request limiting your stack already offers before adding more.

**Step 9. Protect the heavy pages.** Attackers love search pages, login forms, cart pages and any URL that triggers database work. Enable page caching for public pages, add a CAPTCHA or rate limit on login and search, and keep your CMS and plugins updated. A cached page costs the server almost nothing, so a flood against it does far less damage.

**Step 10. Consider a reverse proxy or CDN in front.** Putting a proxy or CDN service in front of your site hides your real server IP and absorbs a lot of junk traffic before it reaches you. If you do this, keep your origin IP private and allow web traffic to your server only from the proxy's addresses.

### Part 4: Monitor and Be Ready

**Step 11. Learn what normal looks like.** You cannot spot an attack without a baseline. Note your usual traffic, connections and CPU at different hours. These commands give you a quick view:

```bash
ss -s
ss -nt state syn-recv | wc -l
ss -nt | awk 'NR>1 {print $5}' | cut -d: -f1 | sort | uniq -c | sort -nr | head
```

The last command lists the IPv4 addresses with the most connections. A very large count from a few addresses is worth a closer look.

**Step 12. Keep backups and a recovery path.** Take regular backups and store a copy away from the server. If a compromise ever happens alongside an attack, you can reinstall your operating system for free from your panel and restore from backup.

## What to Do While an Attack Is Happening

Stay calm and work through this order.

| Order | Action |
|---|---|
| 1 | Check whether the problem is bandwidth, connections or CPU using the commands above |
| 2 | Contact YouStable support on live chat or ticket right away and share the time, symptoms and traffic pattern |
| 3 | Block obvious attacking IPs, for example `sudo ufw deny from 203.0.113.10` or a firewalld rich rule |
| 4 | Tighten rate limits temporarily and serve cached pages where possible |
| 5 | After it ends, review logs, adjust your limits and write down what worked |

Support is available 24/7 in Hindi and English, so you reach a senior engineer and not a chatbot when every minute counts. A dedicated account manager also knows your setup, which saves explanation time.

## FAQs

### Does a dedicated server stop DDoS attacks by itself?

No. A dedicated server gives you dedicated resources and full control, which makes it easier to defend, but you still need a firewall, tuning and monitoring. Very large network floods also need upstream filtering, so speak to support about your needs.

### Is a dedicated server safer than a VPS against DDoS?

You are not affected by noisy neighbours, and an attack on another customer will not use your resources. You also get full root access to build your own defences. That said, safe depends on how well you configure it.

### Can I increase my bandwidth during an attack?

Every server ships with a 1 Gbps uplink, and you can scale to 10 Gbps on request. This helps with heavy traffic, but it is not a substitute for proper filtering.

### Do I need Imunify360?

It is optional, and it is billed only when you enable it. It adds a firewall, malware scanner and intrusion detection on top of what you configure yourself, which is useful if you do not want to manage everything by hand.

### Will my data stay in India?

Yes. Your workload, backups and customer data stay inside Indian data centers.

## Conclusion

You cannot stop anyone from trying to attack your server, but you can make it a very poor target. Choose a plan with spare capacity, close unused ports, tune the kernel, rate limit the web layer, keep an eye on your traffic and keep support one message away.

Ready to set up your server? Check all plans on the [YouStable India dedicated server page](https://www.youstable.com/dedicated-servers/india/), or visit [YouStable](https://www.youstable.com/) if you also need hosting or a domain name for your project.
