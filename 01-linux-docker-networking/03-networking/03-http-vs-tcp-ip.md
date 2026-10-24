# HTTP vs TCP/IP

> Phase 1 — Linux, Docker & Networking › Networking

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

**TCP/IP is the networking protocol suite that enables devices to communicate across networks. HTTP is an application-layer protocol used to exchange requests and responses, commonly between web clients and servers.**

They are not competing protocols. They operate at different layers and work together.

When you open `https://example.com`, the application uses HTTP semantics to request a resource. HTTP/1.1 and HTTP/2 typically run over TCP, while HTTP/3 runs over QUIC, which uses UDP as its transport.

A simplified model of the traditional TCP-based web request looks like this:

```text
Application layer: HTTP
        |
Transport layer: TCP
        |
Internet layer: IP
        |
Link layer: Ethernet / Wi-Fi
        |
Physical network: Signals and transmission
```

Each layer has a different responsibility:

- **HTTP:** Defines requests, responses, methods, headers, status codes, and message semantics.
- **TCP:** Provides a reliable, ordered byte stream between communicating endpoints.
- **IP:** Addresses and routes packets between networks.
- **Ethernet/Wi-Fi:** Transports frames across a local network link.

For example, when a browser requests `https://example.com` using HTTP/1.1 over TCP:

1. DNS resolution determines an IP address for the hostname.
2. The client establishes a TCP connection to the server, typically on port `443` for HTTPS.
3. TLS establishes an encrypted session over that TCP connection.
4. The client sends an HTTP request through the TLS connection.
5. The server returns an HTTP response.
6. TCP delivers the byte stream reliably and in order, retransmitting lost data when necessary.

The important distinction is that **HTTP defines what the client and server communicate, while TCP/IP provides the networking mechanisms that carry that communication.**

## Key points

- **HTTP and TCP/IP operate at different layers**

  | Feature | HTTP | TCP/IP |
  |---|---|---|
  | Primary role | Application-level communication | Network communication and transport |
  | Main responsibility | Defines request and response semantics | Addresses, routes, and transports data |
  | Examples | `GET`, `POST`, `200 OK`, `404 Not Found` | IP addressing, TCP connections, packet routing |
  | Typical use | Web applications, APIs, web services | Communication between networked systems |
  | Relationship | Uses an underlying transport | Carries HTTP traffic through the network stack |

  TCP/IP is a protocol suite, not a single protocol. TCP and IP are individual protocols within that suite.

- **TCP provides reliable, ordered delivery**

  TCP establishes a connection and provides a reliable, ordered byte stream. It handles retransmission, sequencing, flow control, and congestion control.

  For example, if TCP segments are lost during transmission, TCP can retransmit the missing data. The receiving application gets the reconstructed byte stream rather than needing to manage packet retransmissions itself.

  TCP does not guarantee that the application will receive a response, that the server will remain available, or that a request will be processed successfully.

- **IP handles addressing and routing**

  IP packets contain source and destination IP addresses. Routers use these addresses to forward packets toward their destinations.

  IP itself does not guarantee delivery, ordering, or retransmission. Those reliability mechanisms are provided by protocols such as TCP when TCP is used.

- **HTTP defines application behavior**

  HTTP describes how clients and servers exchange messages.

  Important HTTP methods include:

  | Method | Typical purpose |
  |---|---|
  | `GET` | Retrieve a resource |
  | `POST` | Submit data or request processing |
  | `PUT` | Create or replace a resource at a known URI |
  | `PATCH` | Partially modify a resource |
  | `DELETE` | Request resource deletion |

  Common status codes include:

  | Status | Meaning |
  |---|---|
  | `200 OK` | Request succeeded |
  | `201 Created` | A resource was created |
  | `301 Moved Permanently` | Resource has a permanent redirect |
  | `400 Bad Request` | Request is invalid |
  | `401 Unauthorized` | Authentication is required or invalid |
  | `403 Forbidden` | The server refuses the request |
  | `404 Not Found` | The requested resource was not found |
  | `500 Internal Server Error` | The server encountered an internal error |
  | `503 Service Unavailable` | The service is temporarily unable to handle the request |

  TCP cannot interpret these status codes. They belong to the HTTP application protocol.

- **HTTP versions use different transport protocols**

  | HTTP version | Typical transport | Important characteristic |
  |---|---|---|
  | HTTP/1.1 | TCP | Text-based message syntax; persistent connections are supported |
  | HTTP/2 | TCP | Binary framing, multiplexed streams, header compression |
  | HTTP/3 | QUIC over UDP | Multiplexed streams and transport reliability through QUIC |

  HTTP/3 is an important exception to the simplified statement that HTTP always runs over TCP. QUIC provides reliable streams and other transport functions over UDP.

- **HTTP and HTTPS are not the same**

  - HTTP is the application protocol.
  - HTTPS is HTTP secured using TLS.
  - HTTP/1.1 and HTTP/2 commonly use TLS over TCP for HTTPS.
  - HTTP/3 uses QUIC, which incorporates TLS 1.3 security mechanisms.

  Encryption protects data in transit, but HTTPS does not automatically guarantee that an application is secure or that its server is trustworthy.

- **Ports identify application endpoints**

  TCP and UDP use port numbers to identify transport endpoints. IP addresses identify network interfaces or destinations at the network layer.

  Common examples include:

  - HTTP: TCP port `80`
  - HTTPS using HTTP/1.1 or HTTP/2: TCP port `443`
  - HTTPS using HTTP/3: typically UDP port `443`

  These are conventional ports. Services can be configured to use other ports.

- **A TCP connection can succeed while HTTP fails**

  A successful TCP connection only demonstrates that a transport connection was established. The HTTP request can still fail because of authentication, invalid paths, application errors, or other conditions.

  For example:

  ```text
  TCP connection established
           |
           v
  HTTP GET /missing-page
           |
           v
  HTTP/1.1 404 Not Found
  ```

  The network connection worked, but the requested resource was not found.

- **HTTP success does not mean the entire application workflow succeeded**

  An HTTP `200 OK` response indicates success according to the server's HTTP response. The response body may still contain a business-level error, depending on the application's API design.

  Similarly, an HTTP error response can be returned over a perfectly functioning TCP connection.

- **Connection establishment and request processing are separate**

  A simplified TCP connection establishment uses the three-way handshake:

  ```text
  Client                         Server
    |                               |
    | -------- SYN ---------------> |
    | <------- SYN-ACK ------------ |
    | -------- ACK ---------------> |
    |                               |
    |       TCP connection          |
    |                               |
    | -------- HTTP request ------> |
    | <------- HTTP response -----  |
  ```

  The handshake establishes the TCP connection. The HTTP request and response occur afterward.

- **Relevance to Docker and Kubernetes**

  In containerized applications, each networking layer still has its own responsibility.

  Example:

  ```text
  Browser
     |
     | HTTP request
     v
  Host port 8080
     |
     | Docker port mapping
     v
  Container port 80
     |
     v
  Web application
  ```

  Docker port mapping forwards network traffic. It does not transform HTTP into TCP or replace HTTP request handling.

  In Kubernetes, a Service routes traffic to eligible backend Pods. The application protocol and transport protocol still determine how the client and server communicate.

## Commands / manifests

**1. Send an HTTP request with `curl`**

```bash
curl -v http://example.com
```

The `-v` option displays connection and HTTP details, including the resolved destination, connection attempt, request headers, and response headers.

This command typically uses HTTP over TCP because the URL uses HTTP and the server's negotiated or selected protocol will depend on the client and server configuration.

Inspect only the response headers:

```bash
curl -I http://example.com
```

The `-I` option requests headers using an HTTP `HEAD` request. Some servers handle `HEAD` differently from `GET`, so this is not always equivalent to fetching the full resource.

**2. Inspect an HTTPS request**

```bash
curl -v https://example.com
```

Look for details such as:

- The resolved IP address.
- The destination port, typically `443`.
- TLS handshake information.
- The HTTP request and response.
- The returned HTTP status code.

For a TCP-based HTTPS connection, TCP establishes the connection before TLS and HTTP data exchange. With HTTP/3, the transport flow differs because QUIC runs over UDP.

**3. Test TCP connectivity independently of HTTP**

On Linux, macOS, or a suitable environment with Netcat installed:

```bash
nc -vz example.com 443
```

This attempts to establish a TCP connection to port `443`.

A successful connection does not prove that HTTPS or HTTP requests will succeed. It only tests TCP connectivity to the selected address and port.

To test port `80`:

```bash
nc -vz example.com 80
```

If Netcat is unavailable, use another appropriate connectivity tool for the operating system.

**4. Inspect TCP connections on Linux**

```bash
ss -tn
```

This displays TCP sockets and their states.

To include listening TCP sockets:

```bash
ss -ltn
```

To inspect established TCP connections:

```bash
ss -tn state established
```

These commands help investigate transport-level connectivity. They do not show HTTP status codes or explain application-level errors.

**5. Resolve a hostname before testing connectivity**

```bash
getent hosts example.com
```

Or use:

```bash
dig example.com A
```

Then test connectivity:

```bash
nc -vz example.com 443
```

Finally, test the application protocol:

```bash
curl -v https://example.com
```

This sequence helps separate DNS problems, TCP connection failures, and HTTP/TLS problems.

**6. Test HTTP through a Docker port mapping**

Start NGINX:

```bash
docker run -d --name web -p 8080:80 nginx:alpine
```

Send an HTTP request:

```bash
curl -v http://localhost:8080
```

Inspect the port mapping:

```bash
docker port web
```

Inspect the container logs:

```bash
docker logs web
```

Here, Docker forwards host port `8080` to container port `80`. HTTP defines the request and response, while TCP carries the connection for this ordinary HTTP setup.

Clean up:

```bash
docker rm -f web
```

**7. Compare transport failure with HTTP failure**

First, attempt to connect to a service that is not listening:

```bash
curl -v http://127.0.0.1:65534
```

This will normally fail to establish a TCP connection because the chosen port is not listening, unless something else is using that port.

Next, request a nonexistent path from NGINX:

```bash
docker run -d --name web -p 8080:80 nginx:alpine
curl -i http://localhost:8080/nonexistent
```

NGINX will normally return an HTTP `404 Not Found` response.

The distinction:

- A connection refusal is a transport or connection-establishment failure.
- A `404` is an HTTP response received over a functioning connection.

Clean up:

```bash
docker rm -f web
```

**8. Inspect the HTTP status code**

```bash
curl -sS -o /dev/null -w '%{http_code}\n' https://example.com
```

This prints the HTTP response status code while discarding the response body.

A result such as `200` indicates an HTTP success response. Other codes may indicate redirects, client errors, or server errors.

**9. Test an HTTP/3 request when supported**

```bash
curl --http3 -v https://example.com
```

This requires a `curl` build with HTTP/3 support and a destination that supports HTTP/3. The request may fail if the client lacks support, the server does not support HTTP/3, or network policies block the required UDP traffic.

Check the client's supported features:

```bash
curl --version
```

HTTP/3 commonly uses UDP port `443` and QUIC rather than TCP for the transport.

## Common failures & fixes

- **Problem: `curl` reports `Connection refused`.**
  - **Cause:** The destination is reachable, but the connection was rejected because no service is listening on the target port or a network component actively rejected it.
  - **Fix:** Verify the destination port, application status, container port mapping, and listening sockets with `ss -ltn` or `docker port`.

- **Problem: `curl` hangs and eventually times out.**
  - **Cause:** Traffic may be dropped by a firewall, routed incorrectly, or sent to an unreachable destination. The server may also be overloaded.
  - **Fix:** Check DNS resolution, routing, firewall rules, security groups, and the target service. Use `nc -vz HOST PORT` to help isolate TCP connectivity.

- **Problem: TCP connectivity works, but the HTTP request returns `404`.**
  - **Cause:** The server accepted the connection, but the requested resource was not found at that path.
  - **Fix:** Check the URL path, reverse-proxy routing, application routes, and server configuration.

- **Problem: HTTP returns `401` or `403`.**
  - **Cause:** The request lacks valid authentication or does not have sufficient authorization.
  - **Fix:** Check credentials, tokens, permissions, and server access policies. Do not treat this as a TCP connectivity problem.

- **Problem: HTTPS fails with a certificate verification error.**
  - **Cause:** The certificate may be expired, issued for a different hostname, or signed by an untrusted certificate authority. The local clock can also cause validation failures.
  - **Fix:** Verify the hostname, certificate chain, system time, and trusted CA configuration. Avoid using `curl -k` as a permanent solution because it disables certificate verification.

- **Problem: DNS resolution fails before the TCP connection is attempted.**
  - **Cause:** The hostname cannot be resolved through the configured name-resolution path.
  - **Fix:** Inspect `getent hosts`, `dig`, `/etc/resolv.conf`, and the active DNS configuration. Test a known-good hostname to determine whether the issue is domain-specific or more general.

- **Problem: Docker publishes the correct port, but the application is unreachable.**
  - **Cause:** The application may be listening on the wrong container port, bound only to `127.0.0.1` inside the container, or not running.
  - **Fix:** Check `docker ps`, `docker port`, and `docker logs`. Verify that the application listens on the intended container port and, where appropriate, on `0.0.0.0`.

- **Problem: `nc` succeeds, but `curl` fails.**
  - **Cause:** TCP connectivity exists, but TLS negotiation, HTTP behavior, authentication, or application processing may be failing.
  - **Fix:** Run `curl -v` and examine the TLS and HTTP details. Test the correct protocol, hostname, path, and credentials.

- **Problem: A client receives `502 Bad Gateway` or `504 Gateway Timeout`.**
  - **Cause:** A reverse proxy or gateway cannot obtain a valid response from its upstream service, or the upstream does not respond within the expected time.
  - **Fix:** Check upstream DNS, service endpoints, port mappings, application health, and proxy timeout settings. Determine whether the failure occurs between the client and proxy or between the proxy and upstream.

- **Problem: The team assumes HTTP and TCP are interchangeable.**
  - **Cause:** Both are involved in many web requests, but they solve different problems.
  - **Fix:** Troubleshoot in layers: DNS resolution, IP reachability, transport connectivity, TLS where applicable, and then HTTP status and application behavior. Remember that HTTP/3 uses QUIC over UDP rather than TCP.

## References

- [MDN: An overview of HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP/Overview)
- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [RFC 9293: Transmission Control Protocol (TCP)](https://www.rfc-editor.org/rfc/rfc9293)
- [RFC 791: Internet Protocol](https://www.rfc-editor.org/rfc/rfc791)
- [RFC 9114: HTTP/3](https://www.rfc-editor.org/rfc/rfc9114)
- [Docker documentation: Networking](https://docs.docker.com/engine/network/)
- [curl documentation](https://curl.se/docs/)
