7.a. Describe SOCKS5 proxy implementation.
7.b. Describe HTTP/HTTPS proxy implementation.
7.c. Explain limitations of HTTP proxy vs SOCKS5.

## 7.a - SOCKS5 Proxy Implementation

**SOCKS5** (Socket Secure version 5) is a protocol-agnostic proxy that operates at the session layer, forwarding any TCP or UDP traffic. It serves as Tor's primary interface for applications.

### How SOCKS5 Works

SOCKS5 relays network connections through a five-step process:

1. The client establishes a connection to the proxy server.
2. The client and proxy negotiate optional authentication credentials.
3. The client sends the destination address (either an IP address or domain name) along with the port number.
4. The proxy establishes a connection to the specified destination.
5. The proxy forwards data bidirectionally between the client and destination.

SOCKS5 provides several key features that make it versatile. It is protocol-agnostic, meaning it works with any TCP or UDP application without needing to understand the application protocol. The proxy can resolve domain names on behalf of the client, supports optional authentication mechanisms, and handles both IPv4 and IPv6 addresses.

### SOCKS5 in Tor

Tor runs a SOCKS5 proxy server on localhost, typically using port 9050 for the Tor daemon or port 9150 for Tor Browser. When an application wants to use Tor, it follows this connection flow:

1. The application connects to `127.0.0.1:9050`.
2. Tor receives the SOCKS5 request containing the destination address.
3. Tor builds a three-hop circuit consisting of a guard relay, middle relay, and exit relay.
4. The exit relay resolves the domain name (if applicable) and establishes a connection to the destination.
5. Data is relayed through the circuit with encryption applied at each hop.

Tor uses SOCKS5 for several important reasons. First, it provides **DNS privacy** by having the exit relay perform DNS resolution, which prevents the user's ISP from seeing DNS queries. Second, it offers **application flexibility** since any SOCKS5-aware application can use Tor without modification. Third, Tor can implement **circuit isolation** by using the SOCKS5 username field to isolate circuits per application, preventing correlation between different activities.

Tor extends the standard SOCKS5 protocol with additional commands. It supports `RESOLVE` for DNS-only lookups and `RESOLVE_PTR` for reverse DNS queries, going beyond what the standard SOCKS5 specification provides.

The advantages of using SOCKS5 with Tor are significant. It works with any protocol including SSH, FTP, IRC, and custom applications. It prevents DNS leaks by design, introduces minimal overhead, and provides a simple implementation that reduces the attack surface.

## 7.b - HTTP/HTTPS Proxy Implementation

**HTTP proxies** operate at the application layer and are designed specifically for HTTP and HTTPS traffic. Unlike SOCKS5, HTTP proxies understand the HTTP protocol and can inspect, modify, or cache requests.

### How HTTP Proxies Work

For **HTTP traffic**, the proxy parses the incoming request, extracts the destination information, makes a new request to the destination server, and forwards the response back to the client.

For **HTTPS traffic**, HTTP proxies use the `CONNECT` method to create a TCP tunnel. The client sends a `CONNECT example.com:443` request to the proxy. The proxy establishes a connection to the destination and responds with `200 Connection Established`. After this handshake, the proxy forwards encrypted TLS data bidirectionally without inspecting the content.

HTTP proxies provide several features specific to web traffic. They can parse HTTP-specific headers, use CONNECT tunneling for HTTPS connections, manipulate headers to enhance privacy, cache responses to improve performance, and filter content (though only for unencrypted HTTP traffic).

### HTTP Proxy with Tor

Tor does not provide a built-in HTTP proxy server. However, users can employ **Privoxy** as an intermediary layer between applications and Tor:

```
Application → HTTP Proxy (Privoxy:8118) → Tor SOCKS5 (9050) → Tor Network
```

This setup serves several use cases. It provides application compatibility for programs that only support HTTP proxies. Privoxy can scrub identifying headers such as User-Agent, Referer, and cookies before traffic enters the Tor network. It can also perform ad blocking and content filtering to enhance privacy and user experience.

The Tor Browser takes a different approach by using SOCKS5 directly, which is simpler and more efficient. The browser handles privacy protections internally without requiring an intermediate HTTP proxy layer.

## 7.c - Limitations of HTTP Proxy vs SOCKS5

While both proxy types can route traffic through Tor, they have fundamentally different capabilities and limitations.

### Protocol Support

SOCKS5 works with any TCP or UDP protocol, including HTTP, HTTPS, FTP, SSH, SMTP, IRC, gaming protocols, and custom applications. Because it is protocol-agnostic, SOCKS5 simply forwards bytes without needing to understand the application protocol.

HTTP proxies, in contrast, only support HTTP and HTTPS traffic. They cannot proxy SSH, FTP, SMTP, or other non-HTTP protocols. For HTTPS connections, HTTP proxies use CONNECT tunneling, which essentially provides SOCKS-like functionality by creating a transparent TCP tunnel.

### DNS Resolution

SOCKS5 always performs remote DNS resolution. Domain names are sent to the proxy for resolution, which prevents DNS leaks since the ISP never sees the queries. When using Tor, DNS resolution happens at the exit relay.

HTTP proxy DNS behavior depends on the implementation. For HTTP requests, the proxy typically resolves the domain name, which is good for privacy. However, for HTTPS CONNECT requests, some clients might resolve the domain locally before connecting to the proxy, creating a DNS leak risk. When Privoxy is used with Tor, it forwards requests to the SOCKS5 interface, which handles DNS resolution properly.

### Privacy and Security

SOCKS5 provides no content inspection capabilities—it cannot see the data being transmitted, only the destination address. This creates a simple attack surface with fewer potential vulnerabilities. However, SOCKS5 does not perform header scrubbing, so applications must handle their own privacy protections.

HTTP proxies can inspect unencrypted HTTP traffic, which allows them to scrub identifying headers such as User-Agent, Referer, and cookies. They can also filter content to block ads or malicious scripts. However, this inspection capability creates a larger attack surface. For HTTPS connections using CONNECT tunneling, the traffic is encrypted end-to-end and provides privacy equivalent to SOCKS5.

### Application Compatibility

SOCKS5 is supported by most modern applications including browsers, SSH clients, and torrent clients. However, some legacy applications and mobile apps lack SOCKS5 support.

HTTP proxies enjoy nearly universal support across all applications. Most operating systems provide built-in HTTP proxy support, making configuration straightforward. However, HTTP proxies are limited to HTTP and HTTPS protocols.

### Recommendations

**Use SOCKS5 when:**
- You need to proxy non-HTTP protocols such as SSH, FTP, IRC, or gaming traffic.
- You want minimal overhead and maximum performance.
- Your application supports SOCKS5 natively.
- You are using Tor Browser or other privacy-focused applications.

**Use HTTP Proxy when:**
- Your application only supports HTTP proxy configuration.
- You need header scrubbing or content filtering capabilities.
- You are dealing with legacy systems that lack SOCKS5 support.

**For Tor specifically:** Prefer using SOCKS5 directly whenever possible. Only use Privoxy or another HTTP proxy as an intermediary if your application requires it. Avoid HTTP proxies that do not properly handle DNS resolution, as they can leak your DNS queries.

### Comparison Summary

| Feature | SOCKS5 | HTTP Proxy |
|---------|--------|------------|
| Protocol support | Any TCP/UDP | HTTP/HTTPS only |
| DNS resolution | Always remote | Implementation-dependent |
| Performance | Minimal overhead | Higher overhead for HTTP |
| Content inspection | No | Yes (HTTP only) |
| Header scrubbing | No | Yes |
| Application support | Modern apps | Universal |
| Complexity | Simple | Complex |

**Bottom line:** SOCKS5 is more versatile and efficient, making it the better choice for Tor in most scenarios. HTTP proxies are useful for compatibility with applications that lack SOCKS5 support or when header scrubbing is required, but they are limited to HTTP and HTTPS protocols. Use Tor's SOCKS5 interface directly whenever your application supports it.