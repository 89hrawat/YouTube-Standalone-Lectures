# How DNS works?

---
## ⭐ Support the Project  
If this **repository** helps you, give it a ⭐ to show your support and help others discover it! 

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

---

### So what is `www`?

* `www` is technically just a **subdomain**, like `docs.kubernetes.io` or `blog.example.com`.
* Historically, it was used to designate the **web server** portion of a domain, separating it from other services like `ftp.example.com` or `mail.example.com`.
* Today, its use is largely **conventional or stylistic** — some websites prefer `www`, while others omit it entirely.

---

> 🔹 In DNS, `www` may point to the apex domain using a **CNAME record** (or an **ALIAS/ANAME**, depending on the DNS provider).  
> 🔹 However, **redirection is handled at the HTTP layer** — via web server configurations (e.g., Nginx, Apache) or CDN-level rules.

> 💡 Some DNS providers (like AWS Route 53) support **ALIAS records at the apex domain**, which behave like CNAMEs but are permitted at the root — useful for pointing `example.com` directly to load balancers or CDN endpoints.


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


#### **2. Recursive Resolution** *(Green)*

The DNS resolver begins a **recursive lookup** through the DNS hierarchy:

1. It asks a **Root DNS server** (represented as `.`) where to find information about the **`.io`** TLD.
2. The Root server replies with a referral to the **`.io` nameservers**.
3. The `.io` nameservers respond with a referral to the **authoritative nameservers** for `kubernetes.io`.
4. The authoritative nameservers for `kubernetes.io` return the **IP address** (A or AAAA record) for `docs.kubernetes.io`.

> **Note**: A **nameserver** is a DNS server that holds DNS records for a domain. It either gives a referral to another server or directly returns the correct IP. We'll explore more about nameservers shortly.


#### **3. Response & Caching** *(Green)*

The resolver **caches** the resolved IP address for the duration of its **TTL (Time To Live)**.
This helps avoid repeated lookups for popular domains.
It then sends the resolved IP back to your system.

#### **4. Connection Established** *(Pink)*

Your browser uses the IP to initiate a **TCP (or TLS)** connection to the Kubernetes docs server and loads the website.

> ⚡ DNS resolvers often cache popular domains like `kubernetes.io` to improve performance and reduce repeated lookups through the root and TLD servers.

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

### Few Important Terms in DNS


### 1. **DNS Zone**

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

### 📄 Sample DNS Zone File – `kubernetes.io`

```dns
$TTL 3600
@       IN  SOA     ns1.digitalocean.com. admin.kubernetes.io. (
                    2025072401 ; Serial number
                    3600       ; Refresh interval
                    1800       ; Retry interval
                    604800     ; Expire time
                    86400 )    ; Minimum TTL

        IN  NS      ns1.digitalocean.com.     ; Primary nameserver
        IN  NS      ns2.digitalocean.com.     ; Secondary nameserver

        IN  A       3.33.186.135              ; Apex domain: kubernetes.io

www     IN  CNAME   kubernetes.io.            ; Redirect www.kubernetes.io → kubernetes.io
docs    IN  A       52.219.165.51             ; docs.kubernetes.io
blog    IN  A       99.84.214.102             ; blog.kubernetes.io
discuss IN  A       13.226.217.45             ; discuss.kubernetes.io

mail    IN  MX 10   mail.kubernetes.io.       ; Mail exchange server
mail    IN  A       198.51.100.25             ; Mail server IP address
```

---

### Key Fields Explained

| Record  | Meaning                                                               |
| ------- | --------------------------------------------------------------------- |
| `$TTL`  | Default Time To Live — how long records can be cached by resolvers.   |
| `@`     | Refers to the root of this DNS zone — here, `kubernetes.io`.          |
| `SOA`   | Start of Authority — defines key metadata and the primary DNS server. |
| `NS`    | Nameserver entries for this zone — used for DNS delegation.           |
| `A`     | Maps a domain or subdomain to an IPv4 address.                        |
| `CNAME` | Creates an alias — forwards to another domain name.                   |
| `MX`    | Mail exchange record — defines the mail server for the domain.        |

---

### 2. **DNS Name Servers (DNS Servers)**

DNS name servers are specialized systems responsible for responding to DNS queries. They **store DNS zones** and serve the records defined within them (like A, AAAA, CNAME, MX).

DNS servers fall into two key categories:

#### a) **Non-Authoritative Name Servers**

* These servers **do not own the original data**. Instead, they respond using **cached results** from previous lookups.
* They are typically run by your **ISP** (e.g., Jio, BSNL) or **public DNS providers** like Google (`8.8.8.8`) and Cloudflare (`1.1.1.1`).
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

> Some DNS providers (like AWS Route 53 or Cloudflare) act as **both registrar and DNS host**, making them one-stop solutions.



> You can use **different providers for your domain registrar and your DNS hosting**.
> For example, you might **register your domain through GoDaddy**, but **use Amazon Route 53 to manage your DNS and nameservers**.
> This flexibility allows you to keep domain ownership and DNS management separate based on your preferences or technical needs.


---

## Low-Level Understanding of DNS

DNS can be understood as a **hierarchical, inverted tree**, where resolution begins at the **Root (`.`)** and flows down to subdomains like `www.kubernetes.io`. Here's how the pieces work together under the hood:

---

### The Root Zone: The Starting Point of All DNS Queries

* At the top of the DNS hierarchy is the **Root (`.`)** — the starting point for every DNS resolution request.
* There are **13 root IPs**, but they represent a **globally distributed network** of hundreds of servers using **Anycast** for high availability and fault tolerance.
* These root servers serve the **root zone file**, managed by **IANA** under **ICANN**.
* The root zone file is simple: it **does not hold IPs of domains**, but instead **lists the nameservers of all TLDs** like `.com`, `.org`, `.io`, `.net`, etc.
* So, the root is **authoritative for TLDs only** — it helps resolvers find the next hop in the hierarchy.

> Example:
> The root knows which servers are responsible for `.io`, but not for `kubernetes.io` or `docs.kubernetes.io`.

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

## 🌐 How DNS Resolves a Domain Name

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
