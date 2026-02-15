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
podman run --rm -it -p 127.0.0.1:9901:9901 -p 10000:10000 -p 127.0.0.1:10001:10001 -p127.0.0.1:10002:10002 -v $(pwd)/config.yaml:/config.yaml docker.io/envoyproxy/envoy:v1.37-latest --log-level info -c config.yaml
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

The full API is documented in [API.md](API.md)

## Architecture

![Architecture of the config.yaml](img/architecture.png)

TODO: explain the life of a request. In steps. And outline config. (Github links to lines?)

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

Note: To capture traffic with Wireshark, the container needs to be run with `--network host`.
This also disables the portmapping and makes all ports, including the admin interface, available over the network!

#### Performance

FizzBuzz'ing directly at the edge allows to shave off over 10ms of latency and enables serving 10x more requests per second!

#### Benchmarking setup

All measurements were done on my retro 10 year old ThinkPad T460, 16Gb RAM, i5-6300U with speedstep disabled for reproducibility (actually, it seems the skylake CPU has aged, when it clocks down from high GHz speedstep states, the system likes to hard-freeze).

According to ChatGPT, we can compare our solution to a state-of-the-art minimal webserver, using netcat:

```sh
while true; do echo -e "HTTP/1.1 200 OK\r\n..." | nc -l 10000; done
```

But instead of serving static content, we actually need to _compute_ FizzBuzz, so we compare against:

```sh
while true; do python3 -c 'fizzbuzz=",".join("Fizz"*(i%3==0) + "Buzz"*(i%5==0) or str(i) for i in range(1,101));print(f"HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\ncontent-length: {len("FizzBuzz:\n")+len(fizzbuzz)+1}\r\nConnection: close\r\n\r\nFizzBuzz:\n"+fizzbuzz)' | nc -l 10000; done
```

In addition, since all testing was done on localhost, to simulate having to reach a remote backend, we add a 65ms delay via `time.sleep(65/1000)`:

```sh
while true; do python3 -c 'fizzbuzz=",".join("Fizz"*(i%3==0) + "Buzz"*(i%5==0) or str(i) for i in range(1,101));print(f"HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\ncontent-length: {len("FizzBuzz:\n")+len(fizzbuzz)+1}\r\nConnection: close\r\n\r\nFizzBuzz:\n"+fizzbuzz);import time; time.sleep(65/1000)' | nc -l 10000; done
```

In contrast, the Envoy proxy can answer directly from the network edge.

![Benchmarking Setup: FizzBuzz at the Edge Proxy, not the Backend](img/edge.png)

I'm sure this is an absolutely fair setup. :wink:

For fairness :wink:, since above server is single-threaded, we only benachmark with one thread

Our benchmarking tool is

```sh
wrk --threads 1 http://127.0.0.1:10000/
```

|                         | nc-python3     | Envoy-FizzBuzz | Speedup     |
| ----------------------- | -------------- | -------------- | ----------- |
| Reqests / Second (avg)  | 11.08          | 150.40         | **> 10X**   |
| Latency (avg)           | 82.32ms        | 66.50ms        | **> 10ms**  |

All benchmarking data is available in [Benchmark.md](Benchmark.md).
