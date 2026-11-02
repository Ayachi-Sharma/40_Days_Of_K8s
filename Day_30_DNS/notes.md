# DNS (Domain Name System) – Complete Documentation

## 1. What is DNS?

DNS stands for Domain Name System.

It is a naming system that translates human-readable domain names into IP addresses that computers use to communicate over a network.

For example:

When we enter `www.google.com` in a browser, the computer needs Google's IP address to establish a connection.

DNS performs this translation.

Example:

www.google.com → 142.250.x.x

Without DNS, users would need to remember IP addresses instead of domain names.

---

## 2. How DNS Works (DNS Resolution Process)

DNS resolution is the process of converting a domain name into its corresponding IP address.

### Step-by-Step DNS Resolution

1. User Request:
   The user enters www.example.com in the browser.

2. Browser Cache:
   The browser checks whether it already has the IP address stored in its DNS cache.

3. OS Cache:
   If the browser does not have the result, the operating system checks its DNS cache and local hostname mappings.

4. Recursive DNS Resolver:
   If the result is not available locally, the system sends a DNS query to the configured recursive DNS resolver.

5. Root Name Server:
   The recursive resolver contacts a root name server to find the appropriate TLD name server for the domain extension.

6. TLD Name Server:
   The TLD server directs the resolver to the authoritative name server responsible for the requested domain.

7. Authoritative Name Server:
   The authoritative name server returns the requested DNS record containing the IP address.

8. DNS Response:
   The recursive resolver returns the IP address to the client and caches the response according to its TTL.

9. Connection:
   The browser uses the returned IP address to connect to the destination server.

### DNS Resolution Flow

User → Browser Cache → OS Cache → Recursive DNS Resolver → Root Name Server → TLD Name Server → Authoritative Name Server → IP Address Returned to User.

Note:
If the DNS result is already available in a cache, the resolver may not need to contact the root, TLD, and authoritative name servers again.

---

## 3. DNS Hierarchy

DNS follows a hierarchical structure consisting of three important levels.

### A. Root Name Server

- Root servers are at the top of the DNS hierarchy.
- They are represented by a dot (.).
- They direct DNS resolvers to the appropriate TLD name servers.
- They do not normally provide the final IP address of the requested domain.
- There are 13 named root server identities (A to M), operated through many physical instances worldwide.

### B. TLD (Top-Level Domain) Name Server

TLD servers manage domain extensions.

Examples:
- .com
- .org
- .net
- .in
- .edu

Their responsibility is to direct resolvers to the authoritative name servers responsible for a particular domain.

### C. Authoritative Name Server

- Stores the actual DNS records for a domain.
- Provides authoritative answers to DNS queries.
- Contains records such as A, AAAA, CNAME, NS, and others.

Example:

For www.example.com, the authoritative server can return its corresponding IP address.

---

## 4. DNS Caching

DNS caching means temporarily storing DNS query results so that the same domain does not need to be resolved repeatedly.

### Where Does DNS Caching Happen?

1. Browser DNS Cache
2. Operating System DNS Cache
3. Recursive DNS Resolver Cache

### How Does DNS Caching Work?

Suppose you visit www.google.com for the first time.

1. The DNS resolver performs the DNS lookup.
2. It receives the IP address from the authoritative name server.
3. The resolver stores the result in its cache.
4. When you visit the same domain again, the cached result can be returned without repeating the complete DNS resolution process.

### What is TTL?

TTL stands for Time To Live.

It defines how long a DNS record can remain cached.

Example:

Domain: www.example.com
IP Address: 192.168.1.10
TTL: 300 seconds

This means the DNS result can be cached for 300 seconds (5 minutes).

After the TTL expires, the resolver needs to refresh the record before relying on it again.

### Benefits of DNS Caching

- Reduces DNS lookup time.
- Improves website loading performance.
- Reduces network traffic.
- Reduces the workload on DNS servers.

---

## 5. DNS Record Types

DNS uses different record types to store information about domains.

### A. A Record (Address Record)

- Maps a domain name to an IPv4 address.
- IPv4 addresses use 32 bits.

Example:

example.com → 192.168.1.10

### B. AAAA Record (IPv6 Address Record)

- Maps a domain name to an IPv6 address.
- IPv6 addresses use 128 bits.

Example:

example.com → 2001:db8::1

### C. CNAME Record (Canonical Name)

- Creates an alias for another domain name.
- Points one domain name to another domain name rather than directly to an IP address.

Example:

www.example.com → example.com

The DNS resolver then looks up the target domain to obtain its IP address.

Important:
A CNAME generally cannot coexist with other ordinary DNS records at the same name.

### D. NS Record (Name Server)

- Specifies which name servers are authoritative for a DNS zone.
- Helps identify where DNS information for a domain is managed.
- Used in DNS delegation.

Example:

example.com → ns1.example.com
example.com → ns2.example.com

### Quick Comparison

| Record | Purpose |
|--------|---------|
| A | Domain to IPv4 |
| AAAA | Domain to IPv6 |
| CNAME | Alias to another domain |
| NS | Identifies authoritative name servers |

---

## 6. Important Linux Files for DNS

Linux provides configuration files that help the system resolve domain names.

### A. /etc/hosts

This file contains static hostname-to-IP mappings.

Check the file:

cat /etc/hosts

Edit the file:

sudo nano /etc/hosts

Example contents:

127.0.0.1       localhost
192.168.1.10    myserver
192.168.1.20    database

When an application tries to resolve myserver, the system can use this local mapping instead of querying DNS, depending on the system's name-resolution configuration.

Uses:
- Local development.
- Testing domain mappings.
- Resolving internal hostnames.
- Temporarily overriding DNS resolution.

### B. /etc/resolv.conf

This file contains DNS resolver configuration.

Check the file:

cat /etc/resolv.conf

Example contents:

nameserver 8.8.8.8
nameserver 1.1.1.1
search example.com

Explanation:

1. nameserver:
   Specifies the DNS server that the system should query.

2. search:
   Specifies domain suffixes that may be appended to unqualified hostnames.

For example, if the search domain is example.com, resolving server1 may also try server1.example.com.

Common Public DNS Servers:

Google DNS: 8.8.8.8
Cloudflare DNS: 1.1.1.1
Quad9 DNS: 9.9.9.9

Note:
On many Linux systems, /etc/resolv.conf may be managed automatically by NetworkManager or systemd-resolved. Changes may not persist.

### C. /etc/nsswitch.conf

This file determines the order in which Linux checks different sources when resolving hostnames.

Check the file:

cat /etc/nsswitch.conf

Look for:

hosts: files dns

Explanation:

1. files:
   Checks local hostname mappings in /etc/hosts.

2. dns:
   Queries the configured DNS resolver.

This means Linux checks /etc/hosts first and then queries DNS.

The exact order can differ depending on the Linux distribution and configuration.

---

## 7. Useful DNS Commands in Linux

1. nslookup google.com
   Finds DNS information for a domain.

2. dig google.com
   Performs a detailed DNS lookup.

3. dig google.com A
   Queries the A record (IPv4).

4. dig google.com AAAA
   Queries the AAAA record (IPv6).

5. dig google.com NS
   Queries the NS records.

6. host google.com
   Performs a simple DNS lookup.

7. cat /etc/hosts
   Displays local hostname mappings.

8. cat /etc/resolv.conf
   Displays DNS resolver configuration.

9. getent hosts google.com
   Resolves a hostname using the system's NSS configuration.

### Query a Particular DNS Server

Command:

dig @8.8.8.8 google.com

This sends a DNS query directly to Google's public DNS resolver.

---

## 8. Important Points to Remember

- DNS translates domain names into IP addresses.
- Root servers direct resolvers to TLD servers.
- TLD servers direct resolvers to authoritative name servers.
- Authoritative servers provide DNS records.
- Recursive resolvers perform lookups on behalf of clients and cache responses.
- TTL determines how long DNS responses can be cached.
- A record maps a domain to IPv4.
- AAAA record maps a domain to IPv6.
- CNAME creates an alias for another domain.
- NS identifies authoritative name servers.
- /etc/hosts stores local hostname mappings.
- /etc/resolv.conf specifies DNS resolver configuration.
- /etc/nsswitch.conf controls hostname lookup order.

---

## 9. Interview Questions

1. What is DNS and why do we need it?
2. Explain the complete DNS resolution process.
3. What is the difference between a recursive and authoritative DNS server?
4. What is the role of a root name server?
5. What is a TLD server?
6. What is DNS caching?
7. What is TTL in DNS?
8. What is the difference between A and AAAA records?
9. What is the difference between CNAME and A records?
10. What is the purpose of /etc/hosts and /etc/resolv.conf?
11. What happens when a DNS cache entry expires?
12. What is the difference between /etc/hosts and /etc/nsswitch.conf?

---

## 10. Practical Task

Perform the following commands on your Linux machine:

1. Check the local hostname mappings:

   cat /etc/hosts

2. Check DNS resolver configuration:

   cat /etc/resolv.conf

3. Check hostname resolution order:

   cat /etc/nsswitch.conf

4. Find the IP address of Google:

   nslookup google.com

5. Perform a detailed DNS lookup:

   dig google.com

6. Query the A record:

   dig google.com A

7. Query the AAAA record:

   dig google.com AAAA

8. Query the NS record:

   dig google.com NS

9. Query Google DNS directly:

   dig @8.8.8.8 google.com

This practical exercise will help you understand how DNS works and how to troubleshoot DNS-related issues in Linux and DevOps environments.