# DNS basics

> Phase 1 — Linux, Docker & Networking › Networking

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

DNS (Domain Name System) translates human-readable domain names into information that computers use to communicate, most commonly IP addresses.

For example, when you access `https://example.com`, your computer needs to determine which IP address to contact. DNS helps resolve `example.com` to an IP address, such as an IPv4 or IPv6 address.

Without DNS, you would often need to connect to services using IP addresses directly.

**Basic resolution flow:**

```text
Application requests example.com
             |
             v
Check local caches and configured name sources
             |
             v
Ask a DNS resolver if needed
             |
             v
Resolver obtains the required DNS record
             |
             v
Return the IP address
             |
             v
Application connects to the destination
```

A typical DNS lookup involves several components:

- **DNS client:** The application or operating system requesting a name resolution.
- **Stub resolver:** The component on the client that sends DNS queries to a configured resolver.
- **Recursive resolver:** A DNS server that finds the requested answer on behalf of the client, often using cached records.
- **Root DNS servers:** Direct resolvers toward the appropriate top-level domain servers.
- **TLD DNS servers:** Handle domains such as `.com`, `.org`, and `.in`, directing queries toward authoritative servers.
- **Authoritative DNS servers:** Provide the definitive DNS records for a domain.

A recursive resolver may already have the answer cached. If so, it can return the result without querying the root, TLD, and authoritative servers again.

DNS commonly uses UDP port `53` for queries and responses, with TCP port `53` also used when required, including some large responses and zone transfers. DNS over HTTPS (DoH) and DNS over TLS (DoT) use different transport arrangements.

## Key points

- **DNS resolution is not the same as establishing a connection**

  DNS resolves a name into an address. The application then uses that address to establish a connection, such as a TCP connection for HTTPS.

  A successful DNS lookup does not guarantee that the destination application is running, that the port is open, or that a firewall permits the connection.

- **Common DNS record types**

  | Record | Purpose | Example |
  |---|---|---|
  | `A` | Maps a name to an IPv4 address | `example.com → 192.0.2.10` |
  | `AAAA` | Maps a name to an IPv6 address | `example.com → 2001:db8::10` |
  | `CNAME` | Aliases one DNS name to another | `www.example.com → example.com` |
  | `MX` | Identifies mail servers for a domain | Mail delivery destination |
  | `NS` | Identifies authoritative name servers | Domain DNS delegation |
  | `TXT` | Stores text data, commonly used for domain verification and email authentication | SPF or verification records |
  | `PTR` | Supports reverse DNS lookup from an IP address to a name | Reverse lookup |
  | `SRV` | Identifies a service's hostname and port | Service discovery |

  The addresses in these examples are illustrative.

- **Recursive versus authoritative DNS**

  - A recursive resolver looks up records on behalf of a client and may return cached results.
  - An authoritative server holds the DNS records for the zones it serves.
  - Your laptop usually queries a recursive resolver rather than querying authoritative servers directly.

- **DNS caching and TTL**

  DNS responses may be cached by browsers, operating systems, recursive resolvers, and other components.

  The TTL (Time to Live) indicates how long a DNS record may be cached. A record with a TTL of 300 seconds can generally be cached for up to five minutes.

  Changing a DNS record does not necessarily make every client observe the change immediately because previously cached records may remain valid until their TTL expires.

- **DNS resolution order varies by environment**

  A typical Linux system may consult:
  1. Application-specific caches or configured name-resolution mechanisms.
  2. The operating system's name service configuration, such as `/etc/nsswitch.conf`.
  3. Local mappings in `/etc/hosts`.
  4. Configured DNS resolvers, often specified through `/etc/resolv.conf` or a local resolver service.

  The exact order depends on the operating system, application, and resolver configuration. A browser may also use its own DNS cache or secure DNS service.

- **`/etc/hosts` versus DNS**

  `/etc/hosts` is a local file that maps names to addresses. It can resolve a hostname without a DNS query when the configured name-resolution path consults that file first.

  Example:

  ```text
  192.0.2.10 internal.example.test
  ```

  This mapping affects the local machine only. It does not create a public DNS record.

- **DNS in Docker**

  Containers have their own name-resolution configuration. On user-defined Docker networks, containers can generally resolve one another by container name or network alias.

  Example:

  ```text
  Application container
          |
          | DNS lookup: db
          v
  Docker embedded DNS
          |
          v
  Database container IP
  ```

  An application container can connect to a database container using `db:5432` when both containers share a suitable network, `db` resolves to the database container, and the database listens on port `5432`.

  Container names on a user-defined network are useful for service discovery because the application does not need to hardcode the database container's IP address.

- **DNS in Kubernetes**

  Kubernetes provides cluster DNS for service discovery. Applications typically access a Service using its DNS name instead of a changing Pod IP.

  For example:

  ```text
  postgres.database.svc.cluster.local
  ```

  This is the conventional fully qualified DNS name for a Service named `postgres` in the `database` namespace in a cluster using the `cluster.local` domain. The actual cluster domain may differ.

  Kubernetes DNS generally resolves Service names to Service-related addresses. A normal ClusterIP Service resolves to its cluster IP, while a headless Service can resolve to the addresses of its backing endpoints.

- **DNS and load balancing**

  A DNS name can have multiple address records, and DNS-based systems can return different answers according to configuration, location, or health policies.

  However, DNS alone is not equivalent to a complete load balancer. DNS responses may be cached, and clients do not necessarily distribute requests evenly across returned addresses.

- **DNS and HTTPS**

  DNS helps resolve the destination hostname, but HTTPS also involves TLS negotiation and certificate validation. A successful DNS lookup does not prove that the server's TLS certificate is valid or that the website is trustworthy.

## Commands / manifests

**1. Resolve a domain using `getent`**

```bash
getent hosts example.com
```

This uses the system's configured name-resolution path, which can include `/etc/hosts` and DNS depending on the system configuration.

Example output:

```text
93.184.216.34 example.com
```

The returned address may differ depending on the DNS response, resolver, network, and time.

**2. Query DNS using `dig`**

```bash
dig example.com
```

`dig` queries DNS directly through the configured DNS resolver and displays details such as the response status, question, answer, and query time.

Request only IPv4 records:

```bash
dig example.com A
```

Request IPv6 records:

```bash
dig example.com AAAA
```

Request mail server records:

```bash
dig example.com MX
```

Display only the returned addresses:

```bash
dig +short example.com A
```

If `dig` is unavailable, install the appropriate DNS utilities package for your Linux distribution. On Debian or Ubuntu, it is commonly provided by `dnsutils` or `bind9-dnsutils`, depending on the release.

**3. Query a specific DNS resolver**

```bash
dig @1.1.1.1 example.com
```

This sends the DNS query to the resolver at `1.1.1.1`, provided that resolver is reachable and DNS traffic is permitted.

Compare with another resolver:

```bash
dig @8.8.8.8 example.com
```

Different answers do not automatically indicate a fault. Resolver caches, DNS policies, geographic routing, and record changes can produce different results.

**4. Use `nslookup`**

```bash
nslookup example.com
```

Query a particular record type:

```bash
nslookup -type=MX example.com
```

`nslookup` is commonly available on Linux, macOS, and Windows. Its exact options and output vary by implementation.

**5. Inspect Linux name-resolution configuration**

```bash
cat /etc/resolv.conf
cat /etc/hosts
cat /etc/nsswitch.conf
```

- `/etc/resolv.conf` commonly identifies DNS servers and resolver options.
- `/etc/hosts` contains local hostname mappings.
- `/etc/nsswitch.conf` controls the order in which configured name services are consulted on systems using the Name Service Switch.

On systems using `systemd-resolved`, inspect resolver status with:

```bash
resolvectl status
```

Query a name through the system resolver:

```bash
resolvectl query example.com
```

Not every Linux distribution uses `systemd-resolved`.

**6. Check reverse DNS**

```bash
dig -x 8.8.8.8
```

This queries the reverse DNS namespace for a PTR record associated with the IP address.

Reverse DNS is configured separately from ordinary forward DNS. A PTR record is not guaranteed to exist, and it does not prove ownership of an IP address or domain.

**7. Check DNS response details**

```bash
dig example.com
```

Pay attention to:

- `status`: Indicates whether the DNS query succeeded or encountered an error.
- `ANSWER SECTION`: Contains returned records when an answer is available.
- `SERVER`: Identifies the DNS server that responded.
- `Query time`: Shows how long the query took.
- `TTL`: Indicates the remaining cache lifetime reported for the returned record.

Common response codes include:

- `NOERROR`: The query completed without a DNS error. It may still return no records of the requested type.
- `NXDOMAIN`: The queried domain name does not exist according to the responding DNS server.
- `SERVFAIL`: The server could not successfully complete the lookup.
- `REFUSED`: The server refused to answer the query.

**8. Test DNS from a Docker container**

Start an NGINX container on a user-defined network:

```bash
docker network create app-net

docker run -d \
  --name web \
  --network app-net \
  nginx:alpine
```

Run a temporary diagnostic container on the same network:

```bash
docker run --rm \
  --network app-net \
  busybox \
  nslookup web
```

The hostname `web` should resolve to the container's network address when Docker's embedded DNS is operating correctly.

Test external name resolution from the same network:

```bash
docker run --rm \
  --network app-net \
  busybox \
  nslookup example.com
```

This tests DNS resolution, not HTTP connectivity. A successful DNS response does not guarantee that the destination is reachable.

Clean up:

```bash
docker rm -f web
docker network rm app-net
```

**9. Demonstrate container-to-container DNS**

Create a user-defined network and two containers:

```bash
docker network create demo-net

docker run -d \
  --name web \
  --network demo-net \
  nginx:alpine

docker run --rm \
  --network demo-net \
  busybox \
  wget -qO- http://web
```

The second container resolves `web` through Docker's embedded DNS and sends an HTTP request to port `80`, NGINX's default HTTP port.

This works because both containers share the same user-defined network. Publishing NGINX's port on the host is not required for this container-to-container request.

**10. Inspect DNS configuration inside a container**

```bash
docker run --rm nginx:alpine cat /etc/resolv.conf
```

This shows the resolver configuration visible inside the container. On a user-defined Docker network, the container commonly uses Docker's embedded DNS server.

The displayed configuration can differ depending on the Docker network, platform, and daemon settings.

## Common failures & fixes

- **Problem: `dig example.com` returns `NXDOMAIN`.**
  - **Cause:** The queried name does not exist according to the responding resolver, or the domain's DNS configuration is incorrect.
  - **Fix:** Verify the spelling, query the authoritative DNS configuration if you manage the domain, and compare responses from another trusted resolver. Remember that a missing record of one type can produce `NOERROR` with an empty answer rather than `NXDOMAIN`.

- **Problem: DNS lookup returns `SERVFAIL`.**
  - **Cause:** The resolver could not complete the query. Possible causes include authoritative-server failures, DNSSEC validation problems, delegation errors, or upstream connectivity issues.
  - **Fix:** Query another resolver, inspect the domain's authoritative records, and check DNSSEC and delegation configuration when you manage the domain.

- **Problem: A domain resolves on one machine but not another.**
  - **Cause:** The machines may use different resolvers, caches, VPN configurations, search domains, or local hosts files.
  - **Fix:** Compare `dig`, `getent hosts`, `/etc/resolv.conf`, and `/etc/hosts`. Check whether a VPN or corporate DNS configuration changes name resolution.

- **Problem: DNS changes do not appear immediately.**
  - **Cause:** Old records may remain in resolver or application caches until their TTL expires.
  - **Fix:** Check the authoritative answer and the TTL of cached responses. Wait for the relevant cache lifetime or use an appropriate cache-flushing procedure for the operating system or resolver. Lowering a TTL after a change does not retroactively shorten the lifetime of records already cached with a higher TTL.

- **Problem: A hostname resolves, but the application is unreachable.**
  - **Cause:** DNS resolution and network connectivity are separate. The target service may be stopped, listening on the wrong interface, blocked by a firewall, or using the wrong port.
  - **Fix:** Resolve the name first, then test the destination port with a suitable tool such as `curl` or `nc`. Inspect the application and network configuration separately.

- **Problem: A container cannot resolve another container by name.**
  - **Cause:** The containers may be on different networks, or the name or network alias may be incorrect.
  - **Fix:** Inspect networks with `docker network ls` and `docker network inspect NETWORK_NAME`. Attach both containers to the same suitable user-defined network and use the correct container name or network alias.

- **Problem: A Docker container cannot resolve external domains.**
  - **Cause:** The container may have an incorrect DNS configuration, the host may lack connectivity, or outbound DNS traffic may be blocked.
  - **Fix:** Compare resolution on the host and inside the container. Inspect `/etc/resolv.conf`, Docker daemon DNS settings, VPN behavior, and firewall rules. Configure custom DNS only when necessary and use a resolver reachable from the container's network.

- **Problem: A Docker build fails while resolving a package registry hostname.**
  - **Cause:** The build environment may have DNS or network connectivity problems, or the package registry may be unavailable.
  - **Fix:** Test name resolution and HTTPS connectivity from the relevant build environment. Check VPN, proxy, firewall, and resolver configuration. Do not assume that a successful DNS query guarantees that the package registry is accessible.

- **Problem: A local hostname resolves to an unexpected address.**
  - **Cause:** A stale `/etc/hosts` entry, search-domain configuration, resolver cache, or internal DNS record may override the expected result.
  - **Fix:** Compare `getent hosts hostname` with `dig hostname`, inspect `/etc/hosts`, and check the active resolver configuration. These tools follow different resolution paths, so different results can be informative.

- **Problem: A Kubernetes application cannot resolve a Service name.**
  - **Cause:** The Service may not exist in the expected namespace, cluster DNS may be unhealthy, or the application may use an incorrect DNS name.
  - **Fix:** Verify the Service and namespace, inspect the Pod's DNS configuration, and test resolution from a diagnostic Pod. For example, use `service-name.namespace.svc.cluster.local` when the cluster uses the conventional `cluster.local` domain.

## References

- [Docker documentation: Build context](https://docs.docker.com/build/concepts/context/)
- [Docker documentation: `.dockerignore` files](https://docs.docker.com/build/concepts/context/#dockerignore-files)
- [Docker documentation: Networking](https://docs.docker.com/engine/network/)
- [Kubernetes documentation: DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [ISC BIND documentation: `dig`](https://bind9.readthedocs.io/)
- [RFC 1034: Domain Names, Concepts and Facilities](https://www.rfc-editor.org/rfc/rfc1034)
- [RFC 1035: Domain Names, Implementation and Specification](https://www.rfc-editor.org/rfc/rfc1035)
