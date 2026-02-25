<h1 align="center">Caddy L4</h1>
<p align="center">
    <a href="https://github.com/rios0rios0/caddy-l4/releases/latest">
        <img src="https://img.shields.io/github/release/rios0rios0/caddy-l4.svg?style=for-the-badge&logo=github" alt="Latest Release"/></a>
    <a href="https://github.com/rios0rios0/caddy-l4/blob/main/LICENSE">
        <img src="https://img.shields.io/github/license/rios0rios0/caddy-l4.svg?style=for-the-badge&logo=github" alt="License"/></a>
    <a href="https://github.com/rios0rios0/caddy-l4/actions/workflows/default.yaml">
        <img src="https://img.shields.io/github/actions/workflow/status/rios0rios0/caddy-l4/default.yaml?branch=main&style=for-the-badge&logo=github" alt="Build Status"/></a>
</p>

A fork of **Project Conncept**, an experimental Layer 4 app for [Caddy](https://caddyserver.com/). It facilitates composable handling of raw TCP/UDP connections based on properties of the connection or the beginning of the stream.

With it, you can listen on sockets/ports and express logic such as:

- "Echo all input back to the client."
- "Proxy all the raw bytes to 10.0.3.14:1592."
- "If connection is TLS, terminate TLS then proxy all bytes to :5000."
- "Terminate TLS; then if it is HTTP, proxy to localhost:80; otherwise echo."
- "If connection is TLS, proxy to :443 without terminating; if HTTP, proxy to :80; if SSH, proxy to :22."
- "If the HTTP Host is `example.com` or the TLS ServerName is `example.com`, then proxy to 192.168.0.4."
- "Block connections from these IP ranges: ..."
- "Throttle data flow to simulate slow connections."

Because this is a Caddy app, it can be used alongside other Caddy apps such as the [HTTP server](https://caddyserver.com/docs/modules/http) or [TLS certificate manager](https://caddyserver.com/docs/modules/tls).

## Features

### Matchers

- **layer4.matchers.http** - matches connections that start with HTTP requests. Any [`http.matchers` modules](https://caddyserver.com/docs/modules/) can be used for matching on HTTP-specific properties such as header or path. Note that only the first request of each connection can be used for matching.
- **layer4.matchers.tls** - matches connections that start with TLS handshakes. Any [`tls.handshake_match` modules](https://caddyserver.com/docs/modules/) can be used for matching on TLS-specific properties of the ClientHello, such as ServerName (SNI).
- **layer4.matchers.ssh** - matches connections that look like SSH connections.
- **layer4.matchers.ip** - matches connections based on remote IP (or CIDR range).
- **layer4.matchers.proxy_protocol** - matches connections that start with [HAPROXY proxy protocol](https://www.haproxy.org/download/1.8/doc/proxy-protocol.txt).
- **layer4.matchers.socks4** - matches connections that look like [SOCKSv4](https://www.openssh.com/txt/socks4.protocol).
- **layer4.matchers.socks5** - matches connections that look like [SOCKSv5](https://www.rfc-editor.org/rfc/rfc1928.html).

### Handlers

- **layer4.handlers.echo** - An echo server.
- **layer4.handlers.proxy** - Powerful layer 4 proxy, capable of multiple upstreams (with load balancing and health checks) and establishing new TLS connections to backends. Optionally supports sending the [HAProxy proxy protocol](https://www.haproxy.org/download/1.8/doc/proxy-protocol.txt).
- **layer4.handlers.tee** - Branches the handling of a connection into a concurrent handler chain.
- **layer4.handlers.throttle** - Throttle connections to simulate slowness and latency.
- **layer4.handlers.tls** - TLS termination.
- **layer4.handlers.proxy_protocol** - Accepts the [HAPROXY proxy protocol](https://www.haproxy.org/download/1.8/doc/proxy-protocol.txt) on the receiving side.
- **layer4.handlers.socks5** - Handles [SOCKSv5](https://www.rfc-editor.org/rfc/rfc1928.html) proxy protocol connections.

Like the `http` app, some handlers are "terminal" meaning that they don't call the next handler in the chain. For example: `echo` and `proxy` are terminal handlers because they consume the client's input.

## Installation

The recommended way is to use [xcaddy](https://github.com/caddyserver/xcaddy):

```bash
xcaddy build --with github.com/mholt/caddy-l4
```

Alternatively, to hack on the plugin code, you can clone it down, then build and run like so:

1. Download or clone this repo: `git clone https://github.com/mholt/caddy-l4.git`
2. In the project folder, run `xcaddy` just like you would run `caddy`. For example: `xcaddy list-modules --versions` (you should see the `layer4` modules).

## Configuration

Since this app does not support Caddyfile (yet?), you will have to use Caddy's native JSON format to configure it. The [caddy-json-schema plugin by @abiosoft](https://github.com/abiosoft/caddy-json-schema) can give you auto-complete and documentation right in your editor as you write your config.

## Usage Examples

### Simple Echo Server

```json
{
	"apps": {
		"layer4": {
			"servers": {
				"example": {
					"listen": ["127.0.0.1:5000"],
					"routes": [
						{
							"handle": [
								{"handler": "echo"}
							]
						}
					]
				}
			}
		}
	}
}
```

### Echo Server with TLS Termination

Uses a self-signed cert for `localhost`:

```json
{
	"apps": {
		"layer4": {
			"servers": {
				"example": {
					"listen": ["127.0.0.1:5000"],
					"routes": [
						{
							"handle": [
								{"handler": "tls"},
								{"handler": "echo"}
							]
						}
					]
				}
			}
		},
		"tls": {
			"certificates": {
				"automate": ["localhost"]
			},
			"automation": {
				"policies": [
					{
						"issuers": [{"module": "internal"}]
					}
				]
			}
		}
	}
}
```

### TCP Reverse Proxy with TLS and PROXY Protocol

Terminates TLS on 993, and sends the PROXY protocol header to 1143 through 143:

```json
{
	"apps": {
		"layer4": {
			"servers": {
				"secure-imap": {
					"listen": ["0.0.0.0:993"],
					"routes": [
						{
							"handle": [
								{
									"handler": "tls"
								},
								{
									"handler": "proxy",
									"proxy_protocol": "v1",
									"upstreams": [
										{"dial": ["localhost:143"]}
									]
								}
							]
						}
					]
				},
				"normal-imap": {
					"listen": ["0.0.0.0:143"],
					"routes": [
						{
							"handle": [
								{
									"handler": "proxy_protocol"
								},
								{
									"handler": "proxy",
									"proxy_protocol": "v2",
									"upstreams": [
										{"dial": ["localhost:1143"]}
									]
								}
							]
						}
					]
				}
			}
		}
	}
}
```

### HTTP/TLS Multiplexer

Proxies HTTP to one backend, and TLS to another (without terminating TLS):

```json
{
	"apps": {
		"layer4": {
			"servers": {
				"example": {
					"listen": ["127.0.0.1:5000"],
					"routes": [
						{
							"match": [
								{
									"http": []
								}
							],
							"handle": [
								{
									"handler": "proxy",
									"upstreams": [
										{"dial": ["localhost:80"]}
									]
								}
							]
						},
						{
							"match": [
								{
									"tls": {}
								}
							],
							"handle": [
								{
									"handler": "proxy",
									"upstreams": [
										{"dial": ["localhost:443"]}
									]
								}
							]
						}
					]
				}
			}
		}
	}
}
```

### HTTP Host and TLS SNI Filtering

```json
{
	"apps": {
		"layer4": {
			"servers": {
				"example": {
					"listen": ["127.0.0.1:5000"],
					"routes": [
						{
							"match": [
								{
									"http": [
										{"host": ["example.com"]}
									]
								}
							],
							"handle": [
								{
									"handler": "proxy",
									"upstreams": [
										{"dial": ["localhost:80"]}
									]
								}
							]
						},
						{
							"match": [
								{
									"tls": {
										"sni": ["example.net"]
									}
								}
							],
							"handle": [
								{
									"handler": "proxy",
									"upstreams": [
										{"dial": ["localhost:443"]}
									]
								}
							]
						}
					]
				}
			}
		}
	}
}
```

### SOCKS4/5 Proxy with IP Filtering and Authentication

Forwards SOCKSv4 to a remote server and handles SOCKSv5 directly in Caddy, while only allowing connections from a specific network and requiring a username and password for SOCKSv5:

```json
{
	"apps": {
		"layer4": {
			"servers": {
				"socks": {
					"listen": ["0.0.0.0:1080"],
					"routes": [
						{
							"match": [
								{
									"socks5": {},
									"ip": {"ranges": ["10.0.0.0/24"]}
								}
							],
							"handle": [
								{
									"handler": "socks5",
									"credentials": {
										"bob": "qHoEtVpGRM"
									}
								}
							]
						},
						{
							"match": [
								{
									"socks4": {}
								}
							],
							"handle": [
								{
									"handler": "proxy",
									"upstreams": [
										{"dial": ["10.64.0.1:1080"]}
									]
								}
							]
						}
					]
				}
			}
		}
	}
}
```

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

See [LICENSE](LICENSE) for details.
