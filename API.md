# Envoy-FizzBuzz API

## Public API

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