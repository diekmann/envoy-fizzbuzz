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

T460 i5-6300U CPU @ 2.40GHz speedstep disabled (actually, broken, CPU aged?)

According to ChatGPT, we can compare to a state-of-the-art minimal webserver, using netcat:

```sh
while true; do echo -e "HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\ncontent-length: 423\r\nConnection: close\r\n\r\nFizzBuzz:\n1,2,Fizz,4,Buzz,Fizz,7,8,Fizz,Buzz,11,Fizz,13,14,FizzBuzz,16,17,Fizz,19,Buzz,Fizz,22,23,Fizz,Buzz,26,Fizz,28,29,FizzBuzz,31,32,Fizz,34,Buzz,Fizz,37,38,Fizz,Buzz,41,Fizz,43,44,FizzBuzz,46,47,Fizz,49,Buzz,Fizz,52,53,Fizz,Buzz,56,Fizz,58,59,FizzBuzz,61,62,Fizz,64,Buzz,Fizz,67,68,Fizz,Buzz,71,Fizz,73,74,FizzBuzz,76,77,Fizz,79,Buzz,Fizz,82,83,Fizz,Buzz,86,Fizz,88,89,FizzBuzz,91,92,Fizz,94,Buzz,Fizz,97,98,Fizz,Buzz" | nc -l 10000; done
```

(which is fast!!)

But actually need to compute, ....

```sh
while true; do python3 -c 'fizzbuzz=",".join("Fizz"*(i%3==0) + "Buzz"*(i%5==0) or str(i) for i in range(1,101));print(f"HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\ncontent-length: {len("FizzBuzz:\n")+len(fizzbuzz)+1}\r\nConnection: close\r\n\r\nFizzBuzz:\n"+fizzbuzz)' | nc -l 10000; done
```

For fairness, since above server is also single-threaded, we only benachmark with one thread

wrk --threads 1 <http://127.0.0.1:10000/>

Also add a 65ms delay, to simulate having to reach the backend, while the Envoy proxy can answer directly from the network edge.

```sh
while true; do python3 -c 'fizzbuzz=",".join("Fizz"*(i%3==0) + "Buzz"*(i%5==0) or str(i) for i in range(1,101));print(f"HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\ncontent-length: {len("FizzBuzz:\n")+len(fizzbuzz)+1}\r\nConnection: close\r\n\r\nFizzBuzz:\n"+fizzbuzz);import time; time.sleep(65/1000)' | nc -l 10000; done
```

Envoy

```
$ wrk --threads 1 http://127.0.0.1:10000/
Running 10s test @ http://127.0.0.1:10000/
  1 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    66.50ms   13.96ms  92.86ms   51.93%
    Req/Sec   150.40     26.18   202.00     69.00%
  1500 requests in 10.01s, 843.75KB read
Requests/sec:    149.83
Transfer/sec:     84.28KB
```

```
while true; do python3 -c 'fizzbuzz=",".join("Fizz"*(i%3==0) + "Buzz"*(i%5==0) or str(i) for i in range(1,101));print(f"HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\ncontent-length: {len("FizzBuzz:\n")+len(fizzbuzz)+1}\r\nConnection: close\r\n\r\nFizzBuzz:\n"+fizzbuzz);import time; time.sleep(65/1000)' | nc -l 10000; done
```

```
$ wrk --threads 1 http://127.0.0.1:10000/
Running 10s test @ http://127.0.0.1:10000/
  1 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    82.32ms  343.86us  84.53ms   83.78%
    Req/Sec    11.08      3.15    20.00     89.00%
  111 requests in 10.02s, 55.07KB read
  Socket errors: connect 0, read 224, write 16346, timeout 0
Requests/sec:     11.07
Transfer/sec:      5.49KB
```

More real comparison targets

```
while true; do python3 -c 'fizzbuzz=",".join("Fizz"*(i%3==0) + "Buzz"*(i%5==0) or str(i) for i in range(1,101));print(f"HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\ncontent-length: {len("FizzBuzz:\n")+len(fizzbuzz)+1}\r\nConnection: close\r\n\r\nFizzBuzz:\n"+fizzbuzz)' | nc -l 10000; done
```

```
$ wrk --threads 1 http://127.0.0.1:10000/
Running 10s test @ http://127.0.0.1:10000/
  1 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    16.92ms  411.37us  21.31ms   90.82%
    Req/Sec    41.40      3.49    50.00     86.00%
  414 requests in 10.00s, 205.38KB read
  Socket errors: connect 0, read 627, write 61670, timeout 0
Requests/sec:     41.39
Transfer/sec:     20.54KB
```

(including py startup)

```
while true; do echo -e "HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\ncontent-length: 423\r\nConnection: close\r\n\r\nFizzBuzz:\n1,2,Fizz,4,Buzz,Fizz,7,8,Fizz,Buzz,11,Fizz,13,14,FizzBuzz,16,17,Fizz,19,Buzz,Fizz,22,23,Fizz,Buzz,26,Fizz,28,29,FizzBuzz,31,32,Fizz,34,Buzz,Fizz,37,38,Fizz,Buzz,41,Fizz,43,44,FizzBuzz,46,47,Fizz,49,Buzz,Fizz,52,53,Fizz,Buzz,56,Fizz,58,59,FizzBuzz,61,62,Fizz,64,Buzz,Fizz,67,68,Fizz,Buzz,71,Fizz,73,74,FizzBuzz,76,77,Fizz,79,Buzz,Fizz,82,83,Fizz,Buzz,86,Fizz,88,89,FizzBuzz,91,92,Fizz,94,Buzz,Fizz,97,98,Fizz,Buzz" | nc -l 10000; done
```

```
$ wrk --threads 1 http://127.0.0.1:10000/
Running 10s test @ http://127.0.0.1:10000/
  1 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    64.53us   37.81us 381.00us   80.56%
    Req/Sec   338.50      5.69   353.00     80.00%
  3370 requests in 10.00s, 1.63MB read
  Socket errors: connect 0, read 5559, write 193479, timeout 0
Requests/sec:    336.98
Transfer/sec:    167.17KB
```

also, `python3 -m http.server 10000` which serves index.html.

```
 wrk --threads 1 http://127.0.0.1:10000/
Running 10s test @ http://127.0.0.1:10000/
  1 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     4.92ms    1.45ms  27.95ms   75.92%
    Req/Sec     1.39k    38.04     1.44k    95.00%
  13872 requests in 10.00s, 8.06MB read
Requests/sec:   1387.07
Transfer/sec:    824.93KB
```

Also, `podman run --rm -it -p 10000:80 -v $(pwd)/site:/usr/share/nginx/html:ro docker.io/nginx:alpine` is fast!

```
wrk --threads 1 http://127.0.0.1:10000/
Running 10s test @ http://127.0.0.1:10000/
  1 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     1.07ms  775.20us  21.30ms   93.51%
    Req/Sec     9.51k     1.12k   12.15k    71.00%
  94589 requests in 10.00s, 59.63MB read
Requests/sec:   9458.41
Transfer/sec:      5.96MB
```

same with actual 65ms delay in both directions
`sudo tc qdisc add dev lo root netem delay 65ms`

```
wrk --threads 1 http://127.0.0.1:10000/
Running 10s test @ http://127.0.0.1:10000/
  1 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   131.20ms  306.74us 132.89ms   76.80%
    Req/Sec    83.67     24.08   101.00     68.00%
  750 requests in 10.04s, 484.13KB read
Requests/sec:     74.69
Transfer/sec:     48.21KB
```
