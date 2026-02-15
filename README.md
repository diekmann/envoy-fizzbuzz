# Envoy-FizzBuzz

FizzBuzz in the Envoy Proxy.

## Background

The [Envoy Proxy](https://www.envoyproxy.io) is an industry-embraced network proxy.
It can be used to create a [service mesh](https://www.envoyproxy.io/docs/envoy/latest/intro/deployment_types/service_to_service) and/or as [edge proxy](https://www.envoyproxy.io/docs/envoy/latest/intro/deployment_types/double_proxy).

FizzBuzz is an industry-grade algorithmic problem, apparently in need by major companies.

Envoy-FizzBuzz brings FizzBuzz to the Envoy Proxy, enabling the FizzBuzz computation right at the network.
This enables all applications in a service mesh to seamlessly leverage FizzBuzz technology, as well as bringing FizzBuzz directly to the network edge, closer to the customer.

## Running and Deployment

With the `config.yaml` of this repository, a genuine vanilla upstream Envoy container image is sufficient for Envoy-FizzBuzz:

```sh
podman run --rm -it --network host -p 127.0.0.1:9901:9901 -p 10000:10000 -p 127.0.0.1:10001:10001 -p127.0.0.1:10002:10002 -v $(pwd)/config.yaml:/config.yaml docker.io/envoyproxy/envoy:v1.37-latest --log-level info -c config.yaml
```

For production usage, please note the security-hardening of the container runtime:
Only the FizzBuzz application port 10000 is exposed over the network; internal listeners and the admin interface are limited to localhost.

## Usage

```sh
curl 127.0.0.1:10000
```

```data
FizzBuzz:
1,2,Fizz,4,Buzz,Fizz,7,8,Fizz,Buzz,11,Fizz,13,14,FizzBuzz,...
```

Or in the browser directly:

![FizzBuzz shown in the Firefox browser](img/fizzbuzz_firefox.png)

### Public API

The main application port is 1000.
We accept plain HTTP `GET /` requests for a normal FizzBuzz.

Both HTTP/1.1 ...

```sh
$ curl -v 127.0.0.1:10000
*   Trying 127.0.0.1:10000...
* Connected to 127.0.0.1 (127.0.0.1) port 10000
* using HTTP/1.x
> GET / HTTP/1.1
> Host: 127.0.0.1:10000
> User-Agent: curl/8.14.1
> Accept: */*
> 
* Request completely sent off
< HTTP/1.1 200 OK
< content-type: text/plain
< content-length: 423
< date: Sun, 15 Feb 2026 13:34:37 GMT
< server: envoy
< x-envoy-upstream-service-time: 22
< 
FizzBuzz:
1,2,Fizz,4,Buzz,Fizz,7,8,Fizz,Buzz,11,Fizz,13,14,FizzBuzz,...
```

... and HTTP/2 are supported:

```sh
$ curl -v --http2 --http2-prior-knowledge 127.0.0.1:10000
Warning: Overrides previous HTTP version option
*   Trying 127.0.0.1:10000...
* Connected to 127.0.0.1 (127.0.0.1) port 10000
* using HTTP/1.x
* [HTTP/2] [1] OPENED stream for http://127.0.0.1:10000/
* [HTTP/2] [1] [:method: GET]
* [HTTP/2] [1] [:scheme: http]
* [HTTP/2] [1] [:authority: 127.0.0.1:10000]
* [HTTP/2] [1] [:path: /]
* [HTTP/2] [1] [user-agent: curl/8.14.1]
* [HTTP/2] [1] [accept: */*]
> GET / HTTP/2
> Host: 127.0.0.1:10000
> User-Agent: curl/8.14.1
> Accept: */*
> 
* Request completely sent off
< HTTP/2 200 
< content-type: text/plain
< content-length: 423
< date: Sun, 15 Feb 2026 13:33:45 GMT
< server: envoy
< x-envoy-upstream-service-time: 21
< 
FizzBuzz:
1,2,Fizz,4,Buzz,Fizz,7,8,Fizz,Buzz,11,Fizz,13,14,FizzBuzz,1...
```

It is possible to also specify the the header `x-foo` to start FizzBuzz at any number below 100:

```sh
$ curl -v --header 'x-foo: 42' 127.0.0.1:10000
*   Trying 127.0.0.1:10000...
* Connected to 127.0.0.1 (127.0.0.1) port 10000
* using HTTP/1.x
> GET / HTTP/1.1
> Host: 127.0.0.1:10000
> User-Agent: curl/8.14.1
> Accept: */*
> x-foo: 42
> 
* Request completely sent off
< HTTP/1.1 200 OK
< content-type: text/plain
< content-length: 254
< date: Sun, 15 Feb 2026 13:20:09 GMT
< server: envoy
< x-envoy-upstream-service-time: 19
< 
FizzBuzz:
43,44,FizzBuzz,46,47,Fizz,49,Buzz,Fizz,52,53,Fizz,Buzz,56,...
```

The FizzBuzz count stops at 100.
Though, it is possible to request a FizzBuzz for individual larger numbers:

```sh
curl --header 'x-foo: 9999' 127.0.0.1:10000
FizzBuzz:
Buzz
```

Negative numbers technically work, ....

```sh
$ curl --header 'x-foo: -9' 127.0.0.1:10000
FizzBuzz:
-8,-7,Fizz,Buzz,-4,Fizz,-2,-1,FizzBuzz,1,2,Fizz,4,Buzz,...
```

..., but since we iterate till 100, they may easily overwhelm Envoy, ...

```sh
curl --header 'x-foo: -9999' 127.0.0.1:10000
upstream connect error or disconnect/reset before headers. reset reason: overflow
```

Non-numbers are not supported.
They get forwarded to the internal listeners, where they frighten the Lua filters.

```sh
curl --header 'x-foo: x' 127.0.0.1:10000
upstream connect error or disconnect/reset before headers. reset reason: remote reset
```

```data
[2026-02-15 13:27:12.524][11][error][lua] [source/extensions/filters/common/lua/lua.cc:32] script log: [string "function envoy_on_request(handle)..."]:9: attempt to perform arithmetic on local 'n' (a string value)
```

To harden the service against malicious incoming traffic, it is recommended to replace the `request_headers_to_add` `append_action: ADD_IF_ABSENT` with `append_action: OVERWRITE_IF_EXISTS_OR_ADD` to always force a clean start at 0, preventing untrusted data to reach the internal listeners.
This is at the cost of reduced flexibility, no longer permitting a user to specify the start of the FizzBuzz.
An enabled-by-default technical fix would be simple, but this would mean dropping those security considerations from the docs, reducing the enterprise-grade vibes.

### Internal API

The envoy admin interface is reachable via port 9901 on localhost.
The internal ports are reachable on 10001 and 10002, but only on localhost for debugging.
It's easy to cause Lua errors by speaking to the internal ports.

## Architecture

![Architecture of the config.yaml](img/architecture.png)

### Internals

#### HTTP/2 Upgrade

Using Wireshark, we can see that one end2end FizzBuzz requests results in a total of 717 packets.
This is because all internal requests default to HTTP/1.1.
Which means, each internal `x-foo` iteration results in a full TCP 3 way handshake.
Then, the `GET / HTTP/1.1` packet, followed by a TCP `ACK`, followed by passing through the `HTTP/1.1 200` reply with the payload, followed by a TCP `ACK`.
That is a total of 7 loopback packets for each of the 100 internal iterations of FizBuzz.

![Screenshot of Wireshark, showing the 7 packets for one internal recursion](img/wireshark_http11_recursion_annotated.png)

With the internal listeners upgraded to HTTP/2, we are down to 333 packets in total.
This is because the recursive self-requests are now done over the same TCP connection, resulting in "only" 303 packets total.

![Screenshot of Wireshark, shoing a snippet of the single internal recursion TCP connection](img/wireshark_http2_recursion_annotated.png)

Trying to upgrade further, to HTTP/3, would likely be a step backward, since HTTP/3 mandates encryption, which is pointless overhead for localhost connections.
We could tune our setup further, by replacing the localhost TCP connection with a Unix Domain Socket.
But we would lose a bit of the networking vibes here.
