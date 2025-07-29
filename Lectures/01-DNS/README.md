# DNS Explained: What It Is and How It Works

## Video reference for this lecture:

[![Watch the video](https://img.youtube.com/vi/hXFciOpUuOY/maxresdefault.jpg)](https://www.youtube.com/watch?v=hXFciOpUuOY&ab_channel=CloudWithVarJosh)


---
## ⭐ Support the Project  
If this **repository** helps you, give it a ⭐ to show your support and help others discover it! 

---

## Table of Contents

- [Introduction](#introduction)  
- [Understanding DNS: Why We Need It and How It Works](#understanding-dns-why-we-need-it-and-how-it-works)  
  - [Why Do We Need DNS?](#why-do-we-need-dns)  
  - [What is DNS?](#what-is-dns)  
  - [Breakdown of `www.kubernetes.io.`](#breakdown-of-wwwkubernetesio)  
  - [What Happens When You Type `docs.kubernetes.io`](#what-happens-when-you-type-docskubernetesio)  
  - [How to Check Your DNS Server (on macOS)](#how-to-check-your-dns-server-on-macos)  
- [Few Important Terms in DNS](#few-important-terms-in-dns)  
  - [DNS Zone & Zone File](#1-dns-zone--zone-file)  
    - [How DNS Zones and Zone Files Work](#how-dns-zones-and-zone-files-work)  
    - [Sample DNS Zone File — `kubernetes.io`](#-sample-dns-zone-file--kubernetesio)  
  - [DNS Name Servers (DNS Servers)](#2-dns-name-servers-dns-servers)  
  - [Domain Registry](#3-domain-registry)  
  - [Domain Registrar](#4-domain-registrar)  
- [Domain Registrar vs DNS Hosting Provider: Who Does What?](#domain-registrar-vs-dns-hosting-provider-who-does-what)  
- [Low-Level Understanding of DNS](#low-level-understanding-of-dns)  
  - [The Root Zone: Foundation of the Global DNS System](#the-root-zone-foundation-of-the-global-dns-system)  
  - [Who Manages the Root Zone?](#who-manages-the-root-zone)  
  - [The Root Servers: Distributed for Resilience](#the-root-servers-distributed-for-resilience)  
  - [TLDs: Managed by Domain Registries](#tlds-managed-by-domain-registries)  
- [How Domain Registration Works (Step-by-Step Breakdown)](#how-domain-registration-works-step-by-step-breakdown)  
- [How DNS Resolves a Domain Name](#how-dns-resolves-a-domain-name)  
- [Conclusion](#conclusion)  
- [References](#references)  

---

## Introduction

The **Domain Name System (DNS)** is the backbone of how we navigate the internet. It translates human-readable domain names like `kubernetes.io` into machine-usable IP addresses such as `3.33.186.135`. This guide offers a comprehensive breakdown of how DNS works—from the root zone to TLDs, authoritative nameservers, and real-world query resolution paths. Whether you're a curious learner or a cloud engineer, this resource will help you deeply understand the lifecycle of a DNS request, the difference between registrars and DNS hosts, and how systems like Route 53, Cloudflare, or GoDaddy interact behind the scenes.

---

## Understanding DNS: Why We Need It and How It Works

### Why Do We Need DNS?

Humans are good at remembering names, but not IP addresses. You might easily recall `kubernetes.io`, but not its actual IP address like `3.33.186.135`. Computers, on the other hand, need IP addresses to route traffic. DNS (Domain Name System) bridges this gap by translating human-friendly domain names into machine-friendly IP addresses.

This system works behind the scenes every time you open a website. When you type `kubernetes.io` in your browser, your system contacts a DNS server to resolve this name into an IP address. Once the IP is received, your system can establish a connection with the correct server.

---

### What is DNS?

The **Domain Name System (DNS)** is a hierarchical and distributed naming system used to translate **human-readable domain names** (like `kubernetes.io`) into **machine-usable IP addresses** (like `3.33.186.135`).

DNS acts like the **Internet's phonebook**. When you type a website name into your browser, DNS is responsible for finding the correct IP address associated with that name so your computer can reach the correct server.

At a high level, DNS performs the following roles:

* **Name resolution**: Converts domain names into IP addresses.
* **Abstraction**: Shields users from having to remember numeric IPs.
* **Load balancing and failover**: DNS entries can point to multiple IPs to distribute traffic.
* **Decentralized design**: Operates as a globally distributed system with caching and delegation.

DNS operates via a series of **queries and responses** that navigate through a hierarchy of servers—from the **root DNS servers** all the way down to the **authoritative name servers** responsible for a specific domain.

So, when you type `docs.kubernetes.io` into your browser, it’s DNS that silently takes care of locating the exact server where the Kubernetes documentation is hosted—within milliseconds.

---

### Breakdown of `www.kubernetes.io.`

Let’s dissect the domain name `www.kubernetes.io.`

| Part         | Description                                                                             |
| ------------ | --------------------------------------------------------------------------------------- |
| `.`          | **Root** — The starting point of the DNS hierarchy                                      |
| `io`         | **TLD (Top-Level Domain)** — Represents a domain category (e.g., `.com`, `.edu`, `.io`) |
| `kubernetes` | **Second-Level Domain** — Registered under the `.io` TLD                                |
| `www`        | **Subdomain** — A prefix to the main domain (used for services like web, mail, etc.)    |


* **Root (`.`)**: The apex of the DNS hierarchy. Every fully qualified domain name (FQDN) ends with an implicit or explicit dot, indicating the root.
* **TLD (`io`)**: The top-level domain specifies the category. For instance, `.com` is for commercial use, `.edu` for education, `.io` is commonly used by tech companies.
* **Domain (`kubernetes`)**: This is the registered name under the `.io` TLD.
* **Subdomain (`www`)**: Subdomains help segment services under the same domain. `docs.kubernetes.io` is another subdomain that serves documentation.

> In everyday use, we typically omit the trailing dot (root); for example, we write `www.kubernetes.io`, but the fully qualified domain name (FQDN) is actually `www.kubernetes.io.`. Most browsers and tools automatically append this dot during resolution, so even if you don’t type it, the underlying DNS query uses the full form ending with a `.`

---

### **Typically, when you type `www.kubernetes.io`, it redirects to `kubernetes.io`.**

This redirection is **intentional and explicitly configured by the domain owner** — it is not automatic. Many modern websites serve content from the **apex domain** (`example.com`) and configure `www.example.com` to redirect to it (or vice versa) using web server rules or CDN configurations.


### So what is `www`?

* `www` is technically just a **subdomain**, like `docs.kubernetes.io` or `blog.example.com`.
* Historically, it was used to designate the **web server** portion of a domain, separating it from other services like `ftp.example.com` or `mail.example.com`.
* Today, its use is largely **conventional or stylistic** — some websites prefer `www`, while others omit it entirely.

> In DNS, `www` may point to the apex domain using a **CNAME record** (or an **ALIAS/ANAME**, depending on the DNS provider).  
> However, **redirection is handled at the HTTP layer** — via web server configurations (e.g., Nginx, Apache) or CDN-level rules.


**Summary**: `www` has no special behavior. It’s just another subdomain and must be explicitly defined in DNS and handled in web or CDN configurations if redirection or separate behavior is intended.

---

### What Happens When You Type `docs.kubernetes.io`?

Shwetangi tries to access `docs.kubernetes.io`. Let’s walk through what happens behind the scenes in **4 simple steps**.

In the diagram below:

* **Pink** = User or browser-initiated steps
* **Green** = DNS resolver-initiated steps



#### **1. DNS Query Initiation** *(Pink)*

Your system first checks its **local DNS cache**.
If the record isn’t found, it forwards the query to the configured **DNS resolver**—typically your ISP’s resolver or a public one like **Google (`8.8.8.8`)** or **Cloudflare (`1.1.1.1`)**.


---

#### **2. Recursive Resolution** *(Green)*

Before performing a full recursive lookup, the **DNS resolver checks its own cache** to see if it has a recent answer for `docs.kubernetes.io`. If a valid cached response exists, it returns that immediately and skips the rest of the resolution steps.

If the answer isn’t cached, the resolver begins a **recursive lookup** through the DNS hierarchy:

1. It queries a **Root DNS server** (represented as `.`) to locate the nameservers responsible for the **`.io`** TLD.
2. The Root server replies with a referral to the **`.io` nameservers**.
3. The `.io` nameservers respond with a referral to the **authoritative nameservers** for `kubernetes.io`.
4. The authoritative nameservers for `kubernetes.io` return the **IP address** (A or AAAA record) for `docs.kubernetes.io`.

> **Note**: A **nameserver** is a DNS server that holds DNS records for a domain. It either gives a referral to another server or directly returns the correct IP. We'll explore more about nameservers shortly.

---

#### **3. Response & Caching** *(Green)*

The resolver **caches** the resolved IP address for the duration of its **TTL (Time To Live)**.
This helps avoid repeated lookups for popular domains.
It then sends the resolved IP back to your system.

#### **4. Connection Established** *(Pink)*

Your browser uses the IP to initiate a **TCP (or TLS)** connection to the Kubernetes docs server and loads the website.

> ⚡ DNS resolvers often cache popular domains like `kubernetes.io` to improve performance and reduce repeated lookups through the root and TLD servers.

---

### DNS TTL Behavior and IP Change Strategy

DNS caching is governed by the **TTL (Time to Live)** value specified in a DNS record at the time of lookup. When a resolver (such as your ISP or a public DNS like `8.8.8.8`) receives a response with a TTL of 600 seconds, it will cache that domain-to-IP mapping for **up to 10 minutes**, starting from when the response was received. During this window, even if the origin DNS server updates the IP address, the resolver will continue serving the **stale IP** until the TTL expires. This can lead to **temporary inaccessibility or misrouting** if clients rely on outdated records.

🛠️ **Best practice before an IP change**: Lower the TTL (e.g., to 60 or 120 seconds) **several hours in advance**. This ensures that future lookups receive the shorter TTL, allowing resolvers to refresh the record more quickly once the change is made.  
⚠️ Keep in mind: TTL changes are **not retroactive**, resolvers that cached the record earlier will retain the original TTL until it expires.

---

### How to Check Your DNS Server (on macOS)

To see which DNS servers your system uses:

```bash
scutil --dns
```

To test DNS resolution manually and see the server used:

```bash
dig docs.kubernetes.io
```

You may see an output like:

```
;; SERVER: 192.168.1.1#53(192.168.1.1)
```

This tells you that `192.168.1.1` (your router or ISP’s DNS) is being used as the resolver.

---

## Few Important Terms in DNS


### 1. **DNS Zone & Zone File**

A **DNS zone** is a distinct portion of the DNS namespace managed by a single authority—such as a domain owner, DNS provider, or registrar. It contains all the DNS records necessary to resolve names within that part of the hierarchy.

At the heart of a zone is the **zone file** (or its conceptual equivalent), which acts like a database for that domain. It holds mappings for subdomains and services—such as IP addresses (**A** or **AAAA** records), mail servers (**MX**), aliases (**CNAME**), and more.

**Example**:  
The DNS zone for `kubernetes.io` might include records like:

* `docs.kubernetes.io` → points to an IP via an A record  
* `blog.kubernetes.io` → mapped via a CNAME or A record  
* `discuss.kubernetes.io` → another subdomain managed within the same zone  

> Think of a DNS zone as the **authoritative record-keeping boundary** for a domain and its subdomains.

Additional points:

* A single **nameserver** may host multiple zones (e.g., `example.com`, `kubernetes.io`).
* A domain can be split into **multiple zones** if different parts are delegated to separate teams or providers.
* The **Start of Authority (SOA)** record marks the beginning of the zone and includes metadata like:
  - Primary nameserver
  - Contact email
  - Serial number (used for zone versioning)
  - Refresh/retry/expire intervals for zone transfers

This modular structure allows large domains to be delegated, distributed, and managed efficiently across different DNS infrastructure.


---

### How DNS Zones and Zone Files Work

* DNS resolution starts at the **root zone**, the entry point of the DNS hierarchy, managed by **IANA**. The root zone file contains **NS records for all Top-Level Domains (TLDs)** such as `.com`, `.org`, and `.io`, telling resolvers where to go next. Example:

  ```dns
  io.   NS   a0.nic.io.
  io.   NS   b0.nic.io.
  com.  NS   a.gtld-servers.net.
  com.  NS   b.gtld-servers.net.
  org.  NS   a0.org.afilias-nst.info.
  org.  NS   b0.org.afilias-nst.org.
  ```

* Each TLD (e.g., `.io`) maintains its own **zone file**, typically managed by a dedicated registry — for `.io`, that’s **NIC.IO**. These TLD zone files contain **NS records for second-level domains**, delegating control to their respective authoritative nameservers. Example:

  ```dns
  kubernetes.io.   NS   ns1.digitalocean.com.
  kubernetes.io.   NS   ns2.digitalocean.com.
  cwvj.io.         NS   ns1.godaddy.com.
  cwvj.io.         NS   ns2.godaddy.com.
  ```

  * In this case, `kubernetes.io` is delegated to **DigitalOcean's** nameservers.
  * So when a DNS resolver looks up `kubernetes.io`, it:

    1. Is directed from the **root** to the `.io` nameservers.
    2. Is then referred by `.io` to DigitalOcean’s nameservers, which host the actual DNS records.

* Once at the **authoritative nameservers** (e.g., `ns1.digitalocean.com`), the query hits the domain's **zone file**, which stores the actual resource records — like **A**, **CNAME**, **MX**, etc. Example records from `kubernetes.io`'s zone:

  ```dns
  kubernetes.io.         A     3.33.186.135
  www.kubernetes.io.     CNAME kubernetes.io.
  docs.kubernetes.io.    A     52.219.165.51
  blog.kubernetes.io.    A     99.84.214.102
  discuss.kubernetes.io. A     13.226.217.45
  ```

* Large domains (e.g., `google.com`) may be split into **multiple zones** for delegation, load management, or team separation. This is done via **NS records for subdomains** inside the parent zone:

  ```dns
  maps.google.com.   NS   ns1.maps-dns.google.com.
  mail.google.com.   NS   ns1.mail-dns.google.com.
  ```

  This allows independent control and distribution of DNS responsibilities, improving scalability and reliability.


> **Note:** While **NS records** are the primary content of both **root and TLD zone files** (used to delegate authority to the next level of nameservers), these zone files also typically include a **Start of Authority (SOA) record**, optional **Delegation Signer (DS) records** for DNSSEC, and **glue records** (A/AAAA) when required to prevent circular dependencies in DNS resolution.

> **Note:** In the hierarchical and distributed DNS system, **recursive DNS resolvers** (such as your ISP’s resolver or public resolvers like **Google `8.8.8.8`**, **Cloudflare `1.1.1.1`**, **Quad9 `9.9.9.9`**) perform **caching** to speed up lookups and reduce upstream queries.  
>  
> In contrast, **root**, **TLD**, and **authoritative name servers** do **not cache DNS responses**. They serve **authoritative data** directly from their zone files and do not perform recursion.  
>  
> While some authoritative servers may use internal optimizations, they do not cache or forward queries like resolvers do.



---

### 📄 Sample DNS Zone File — `kubernetes.io`

```dns
$TTL 3600
@       IN  SOA     ns1.digitalocean.com. admin.kubernetes.io. (
                    2025072401 ; Serial number (version of this zone file)
                    3600       ; Refresh (how often secondaries check for updates)
                    1800       ; Retry (retry interval if refresh fails)
                    604800     ; Expire (when to stop using stale data)
                    86400 )    ; Minimum TTL (used for negative responses)

        IN  NS      ns1.digitalocean.com.   ; Primary authoritative nameserver
        IN  NS      ns2.digitalocean.com.   ; Secondary authoritative nameserver

        IN  A       3.33.186.135            ; Apex (naked) domain: kubernetes.io

www     IN  CNAME   kubernetes.io.          ; Alias www → apex domain
docs    IN  A       52.219.165.51           ; docs.kubernetes.io points to IP
blog    IN  A       99.84.214.102           ; blog.kubernetes.io points to IP
discuss IN  A       13.226.217.45           ; discuss.kubernetes.io points to IP

mail    IN  MX 10   mail.kubernetes.io.     ; Mail exchanger for kubernetes.io
mail    IN  A       198.51.100.25           ; A record for the mail server
```

---

### 🧾 Key Fields Breakdown

| Field   | Description                                                                                          |
| ------- | ---------------------------------------------------------------------------------------------------- |
| `$TTL`  | Default cache duration for all records unless individually overridden.                               |
| `@`     | Refers to the apex of the zone (`kubernetes.io` in this case).                                       |
| `SOA`   | Start of Authority — defines the zone's primary server and metadata for replication and DNS hygiene. |
| `NS`    | Authoritative nameservers for the domain — stored at the TLD level, repeated here for redundancy.    |
| `A`     | Maps a name to an IPv4 address.                                                                      |
| `CNAME` | Creates an alias (canonical name); cannot coexist with other record types.                           |
| `MX`    | Specifies mail exchange servers along with priority.                                                 |

---

> 🔹 **Note:**
> Every DNS zone file **must include**:
>
> * A **Start of Authority (SOA)** record — it marks the beginning of the zone and defines key operational metadata (e.g., serial number, refresh/retry intervals).
> * At least one **NS (Name Server)** record — this specifies the authoritative DNS servers responsible for answering queries for this zone.
>
> Without these two records, the DNS zone is incomplete and cannot function. The **NS record** ensures your zone can be reached, and the **SOA record** ensures proper synchronization and replication across secondary nameservers.

### Additional Notes

* The **SOA** record is mandatory and appears **only once** per zone file. It signals the start of the zone and drives synchronization for secondary nameservers.
* **NS records** define the nameservers responsible for this zone — although the actual delegation happens at the parent zone (`.io`), this local listing serves as backup.
* The **Apex A record** (`kubernetes.io`) resolves the base domain, while subdomains like `docs` or `blog` are defined independently.
* The **MX** record must point to a domain name (not an IP), which in turn resolves via an A record.

---

### 2. **DNS Name Servers (DNS Servers)**

DNS name servers are specialized systems responsible for responding to DNS queries. They **store DNS zones** and serve the records defined within them (like A, AAAA, CNAME, MX).

DNS servers fall into two key categories:

#### a) **Non-Authoritative Name Servers**

* These servers **do not own the original data**. Instead, they respond using **cached results** from previous lookups.
* They are typically run by your **ISP** (e.g., Airtel, Jio, AT&T) or **public DNS providers** like Google (`8.8.8.8`) and Cloudflare (`1.1.1.1`).
* Their main purpose is to **speed up lookups** by reducing the need to query authoritative servers repeatedly.


#### b) **Authoritative Name Servers**

* These servers are the **source of truth** for a domain. They hold the actual zone files with DNS records.

* Only authoritative servers are allowed to provide **official, original answers** to queries.

* For example, the authoritative name server for `kubernetes.io` is responsible for answering:

  ```
  What is the IP address of docs.kubernetes.io?
  ```

* If the record exists, this server will respond with it. If not, it may reply with NXDOMAIN (non-existent domain).

---

> Think of **non-authoritative servers** as efficient middlemen that answer from memory, while **authoritative servers** are the authoritative source that defines the rules and values for a given domain.

> **Note**:
> When reading a domain name like `kubernetes.io`, it's helpful to walk **from right to left**:
>
> * `.` (Root) knows where to find the **authoritative nameservers** for each **Top-Level Domain (TLD)** — like `.com`, `.org`, `.io`, etc.
> * Each TLD (like `.io`) in turn knows where to find the **authoritative nameservers** for second-level domains under it — like `kubernetes.io`.
> * The **authoritative nameservers** for `kubernetes.io` hold the actual **DNS records** (like A, CNAME) for subdomains such as `docs.kubernetes.io`.
>
> In this system:
>
> * **Root** and **TLD (Top-Level Domain)** servers **do not store IP addresses** for individual websites.
> * Their job is to **delegate** the query to the appropriate next-level nameserver.
>
> For example:
>
> * The `.com` server redirects to Google’s nameservers for `google.com`
> * The `.org` server redirects to Wikimedia’s nameservers for `wikipedia.org`
> * The `.io` server redirects to DigitalOcean’s nameservers for `kubernetes.io`
>
> The **authoritative nameservers** at the domain level provide the **final answer**—typically an A (IPv4) or AAAA (IPv6) record with the real IP.
>
> Meanwhile, **DNS resolvers** (run by your ISP or public providers like **Google `8.8.8.8`**, **Cloudflare `1.1.1.1`**) **cache** these resolved IPs temporarily to avoid repeated lookups and reduce latency.

---

#### 3. **Domain Registry**

A **domain registry** is the authoritative organization responsible for managing all domain names under a specific **Top-Level Domain (TLD)** — such as `.com`, `.org`, `.io`, etc. It operates and maintains the **central database** that stores ownership, status, and authoritative nameserver information for every domain registered under that TLD.

The registry is also responsible for:

* Maintaining the **TLD-level DNS servers** (e.g., the `.io` nameservers).
* Delegating domains to their **authoritative name servers**, as provided by registrars or domain owners.
* Coordinating with **domain registrars**, who interface with end users to sell and register domain names.

> Think of a domain registry as the **wholesaler** in the domain name system, while registrars (like GoDaddy, Namecheap) act as **retailers**.

**Example**:

* The `.io` TLD is managed by the **Internet Computer Bureau**, and its registry is known as **NIC.IO**.
* The `.com` TLD is managed by **Verisign**, which operates the `.com` registry.

---

#### 4. **Domain Registrar**

A **domain registrar** is a company accredited by a **domain registry** to sell and manage domain names on behalf of end users. When you register a domain like `kubernetes.io`, you're interacting with a **registrar**, not the registry directly.

Registrars serve as the **interface between the user and the registry**, handling:

* **Domain purchases and renewals**
* Configuration of **authoritative name servers**
* Management of **WHOIS records** and domain ownership
* Optional services like **DNS hosting**, **domain privacy**, and **email forwarding**

> While **registries** maintain the master record for each TLD, **registrars** provide the tools and UI for you to control your domain settings.

**Popular Registrars**:

* GoDaddy
* Namecheap
* Google Domains
* AWS Route 53
* Hover, Porkbun, and others

**Example**:
If you register `kubernetes.io` via GoDaddy:

* GoDaddy handles your purchase, WHOIS information, and interface for DNS.
* The `.io` registry (NIC.IO) maintains the official TLD database entry for your domain.
---

### Domain Registrar vs DNS Hosting Provider: Who Does What?

Some platforms like **Cloudflare** or **AWS Route 53** offer both **domain registration** and **DNS hosting**, but these are fundamentally **two separate roles**.

You can register a domain with one provider (e.g., **GoDaddy**) and host its DNS with another (e.g., **Route 53**). This separation gives you flexibility to choose best-in-class services for each role.

When you register a domain with **GoDaddy**, it becomes your **registrar** — maintaining your WHOIS info and pushing nameserver (NS) records to the TLD registry. If you point those nameservers to **Route 53**, GoDaddy updates the `.com` TLD zone to delegate authority. From that point onward, **Route 53 handles all DNS resolution**. GoDaddy is only involved when you register, renew, transfer the domain, or update the NS delegation.

> All DNS queries go: **Root → TLD → Route 53**, which serves A, CNAME, MX, and other records.
> **GoDaddy is not part of the query path once delegation is complete.**

---

| **Responsibility**                      | **Handled By**      |
| --------------------------------------- | ------------------- |
| Domain name registration                | GoDaddy (Registrar) |
| WHOIS and ownership records             | GoDaddy             |
| Pushing NS records to TLD registry      | GoDaddy             |
| DNS zone file and records (A, MX, etc.) | Route 53 (DNS Host) |
| DNS query responses                     | Route 53            |
| Changing NS (delegation)                | GoDaddy Dashboard   |
| Renewing or transferring domain         | GoDaddy             |
| Editing DNS records, TTLs, subdomains   | Route 53 Console    |

---

## Low-Level Understanding of DNS

DNS can be understood as a **hierarchical, inverted tree**, where resolution begins at the **Root (`.`)** and flows down to subdomains like `www.kubernetes.io`. Here's how the pieces work together under the hood:

---

### The Root Zone: Foundation of the Global DNS System

* At the very top of the DNS hierarchy is the **Root Zone (`.`)** — the universal starting point for all DNS resolution processes.
* It contains **no domain-specific records**. Instead, it holds **NS (Name Server) records** for all **Top-Level Domains (TLDs)** like `.com`, `.org`, `.io`, `.net`, etc.
* These NS records help DNS resolvers know *where to go next* in order to resolve domain names.

---

### Who Manages the Root Zone?

* The **Root Zone file** is maintained by **IANA** (Internet Assigned Numbers Authority), which is a function of **ICANN** (Internet Corporation for Assigned Names and Numbers).
* **Verisign** acts as the **Root Zone Maintainer**, meaning it distributes the final signed root zone to all root servers after changes are approved by IANA.
* The content of the root zone file is relatively small — usually just a few megabytes — but it is critically important.

---

### The Root Servers: Distributed for Resilience

* There are **13 named root servers**, labeled **A–M.root-servers.net** — but this does *not* mean there are only 13 machines.
* These 13 root servers are operated by **12 independent organizations** (Verisign operates two: A and J).
* Through **Anycast routing**, each root server label (e.g., `A.root-servers.net`) is actually served by **hundreds of identical instances** deployed across the globe.
* Each of these root server instances **hosts a copy of the root zone file**, ensuring the system remains available even during regional outages or attacks.

> **Note:** Anycast routing allows multiple servers to share the same IP address globally. DNS queries are automatically routed to the nearest or fastest server based on network topology.


---

### TLDs: Managed by Domain Registries

Each **Top-Level Domain (TLD)** — such as `.com`, `.org`, or `.io` — is managed by a **Domain Registry**. Their responsibilities include:

* Maintaining the **zone file for the TLD** (e.g., `.io`)
* Running the **TLD’s authoritative nameservers**
* Storing the **NS (nameserver) delegation records** for all registered domains under that TLD

These TLD nameservers don't store the **actual DNS records (like A or MX)**. Instead, they **delegate** each domain to its **own authoritative nameservers**.

> Examples:
> • `.io` is managed by **NIC.IO**, which knows the nameservers for `kubernetes.io`
> • `.com` is managed by **Verisign**, which delegates domains like `amazon.com`, `facebook.com`
> • `.org` is managed by **Public Interest Registry (PIR)**  

---

## How Domain Registration Works (Step-by-Step Breakdown)

This section explains the behind-the-scenes process of registering a new domain — in this case, **`cwvj.io`**, as done by Varun. The diagram outlines how control flows from the domain owner to the global DNS hierarchy, involving the **registrar**, **registry**, and **root servers**.

---

### Step 1: Domain Owner Initiates Registration

Varun decides to register the domain `cwvj.io`.

* He visits a **domain registrar** like **GoDaddy.com**, which provides a UI to search and purchase domain names.
* Varun completes the purchase, becoming the legal owner of `cwvj.io`.

---

### Step 2: Registrar Handles Domain Registration and Nameserver Setup

The **domain registrar** (e.g., GoDaddy) performs the following actions:

* Officially **registers `cwvj.io`** on Varun's behalf.
* Assigns a set of **default authoritative nameservers** to manage DNS for the domain.

  * Example:

    ```
    ns1.godaddy.com  
    ns2.godaddy.com
    ```
* Sends this **NS (Name Server) information** to the **domain registry** responsible for `.io` domains.

---

### Step 3: Registry Adds the Domain to the TLD Zone File

The **domain registry** (for `.io`, it's **NIC.IO**) receives the NS data and updates the zone file for `.io`:

* It adds an entry for `cwvj.io`:

  ```dns
  cwvj.io.  NS  ns1.godaddy.com.
  cwvj.io.  NS  ns2.godaddy.com.
  ```
* This **delegates authority** to GoDaddy’s nameservers.
* Now, any DNS resolver that queries for `cwvj.io` will be told to ask **ns1/ns2.godaddy.com** for answers.

---

### Step 4: Root Servers Enable TLD Routing (We don't do anything here, this is aready in place)

At the top of the DNS hierarchy lies the **Root Zone** (denoted by a dot `.`). Root servers don’t know about individual domains like `cwvj.io`, but they **do know** where to find the authoritative nameservers for each **Top-Level Domain (TLD)** such as `.io`.

* When a DNS resolver starts resolving `cwvj.io`, it first contacts a **root server**.
* The **root server responds with a referral** to the **`.io` TLD nameservers** (e.g., `a0.nic.io`, `b0.nic.io`).
* This allows the resolver to continue the query down the chain, eventually reaching `ns1.godaddy.com` to get the final answer.

---

### Summary of Each Role

| Role             | Responsibility                                                             |
| ---------------- | -------------------------------------------------------------------------- |
| **Domain Owner** | Initiates domain registration (e.g., Varun registering `cwvj.io`).         |
| **Registrar**    | Facilitates registration, assigns nameservers, and sends info to registry. |
| **Registry**     | Maintains the `.io` zone, adds NS records, and delegates DNS authority.    |
| **Root Servers** | Know where to find TLD servers like `.io`, `.com`, and `.org`.             |

---

### Result: DNS Resolution Path Is Established

After these steps:

* The domain is now officially part of the DNS hierarchy.
* When someone tries to access `https://cwvj.io`, DNS resolvers follow this path:

  1. Ask Root → redirected to `.io` TLD servers.
  2. Ask `.io` TLD → redirected to GoDaddy’s nameservers.
  3. Ask GoDaddy’s nameservers → receive the actual IP address via an A record.

This **multi-layered delegation** is what makes DNS scalable and globally distributed.

---

## How DNS Resolves a Domain Name

### *(Shwetangi opens `https://cwvj.io` in her browser)*

#### Step 1: User Initiates Request  
**Action:** Shwetangi attempts to access `cwvj.io` using a browser or application.  
**Explanation:** This initiates the DNS resolution process to convert the domain name into an IP address that the system can use to locate and communicate with the destination server.

#### Step 2: Computer Checks Local DNS Cache  
**Action:** The operating system inspects its local DNS cache for a valid record of `cwvj.io`.  
**Explanation:** If a non-expired record is found (based on TTL), the system bypasses external DNS queries and uses the cached IP address directly.

#### Step 3: Computer Sends Query to DNS Resolver  
**Action:** If no local cache is available, the computer forwards the DNS query to its configured DNS resolver—typically an ISP resolver or a public resolver like Google DNS or Cloudflare.  
**Explanation:** The resolver acts as an intermediary that performs recursive lookups on behalf of the client.

#### Step 4: Resolver Checks Its Own Cache  
**Action:** The DNS resolver checks its internal cache for a recent record of `cwvj.io`.  
**Explanation:** If a valid cached response exists, the resolver returns it immediately. Otherwise, it begins recursive resolution by querying upstream nameservers.

#### Step 5: Resolver Queries the Root Nameserver  
**Action:** The resolver sends a query to a root nameserver asking which nameservers are responsible for the `.io` top-level domain.  
**Explanation:** Root nameservers are the entry point into the DNS hierarchy. They don’t store domain-specific records but delegate responsibility to TLD nameservers.

#### Step 6: Root Nameserver Responds with TLD Nameservers  
**Action:** The root nameserver replies with a referral to the `.io` TLD nameservers (e.g., `a0.nic.io`, `b0.nic.io`).  
**Explanation:** This referral enables the resolver to continue its query by contacting the appropriate TLD-level servers.

#### Step 7: Resolver Queries the `.io` TLD Nameserver  
**Action:** The resolver asks the `.io` nameserver which authoritative nameservers are responsible for `cwvj.io`.  
**Explanation:** TLD nameservers manage delegation for domains under their zone and respond with authoritative server details for the requested domain.

#### Step 8: TLD Nameserver Responds with Authoritative Nameservers  
**Action:** The `.io` nameserver returns a referral to the authoritative nameservers for `cwvj.io` (e.g., `ns1.godaddy.com`, `ns2.godaddy.com`).  
**Explanation:** These authoritative nameservers hold the actual DNS records for `cwvj.io`, including its A, AAAA, and other resource records.

#### Step 9: Resolver Queries the Authoritative Nameserver  
**Action:** The resolver sends a query to the authoritative nameserver asking for the IP address of `cwvj.io`.  
**Explanation:** This is the final step in the recursive lookup chain. The authoritative server provides the definitive answer for the domain.

#### Step 10: Authoritative Nameserver Responds with A Record  
**Action:** The authoritative nameserver replies with the A record for `cwvj.io`, such as `15.202.3.55`.  
**Explanation:** The A record maps the domain name to its IPv4 address, enabling the client to initiate a network connection.

#### Step 11: Resolver Caches the Result  
**Action:** The DNS resolver stores the A record in its cache for future queries.  
**Explanation:** Caching reduces latency and minimizes repeated upstream queries. The record is retained for a duration specified by its TTL.

#### Step 12: Computer Receives IP Address  
**Action:** The resolver returns the IP address to the computer, which then passes it to Shwetangi’s browser or application.  
**Explanation:** With the IP address resolved, the browser can initiate a TCP/IP connection to the server hosting `cwvj.io`, completing the DNS resolution process.

---


#### Final Step (not numbered in diagram but crucial):

With the IP in hand, the browser:

* Opens a **TCP connection** to the server at `15.202.3.55`
* Performs a **TLS handshake** for HTTPS
* Sends an **HTTP(S) request** to fetch the webpage
* Renders the content to the user


#### Key Concepts Reinforced

* **DNS resolution is recursive** when the resolver doesn’t have cached data
* The system involves **multiple layers**: local cache → resolver → root → TLD → authoritative NS
* Only **authoritative nameservers** store real records like A, MX, CNAME
* The **root and TLD servers never store actual IPs**, only pointers to the next level
* **Caching is vital** to reduce global DNS traffic and latency

---

## Conclusion

DNS is a globally distributed, hierarchical system that ensures websites are reachable using easy-to-remember names rather than IP addresses. It relies on clearly defined roles—**root servers**, **TLD registries**, **authoritative name servers**, and **recursive resolvers**—to delegate and resolve domain queries efficiently. Understanding how queries propagate through DNS layers and how caching, delegation, and authoritative answers work is essential for developers, site owners, and anyone managing domain infrastructure. As we’ve seen, even seemingly simple actions like typing a URL involve multiple DNS components working together in milliseconds to connect you to your destination.

---

## References
1. [Cloudflare Learning Center – What is DNS?](https://www.cloudflare.com/learning/dns/what-is-dns/)
2. [AWS Route 53 Documentation](https://docs.aws.amazon.com/route53/)
---
