# Benchmark Raw Numbers

All tests done on a ThinkPad T460, 16Gb RAM, i5-6300U with speedstep disabled, on localhost.

## Envoy-FizzBuzz

```sh
podman run --rm -it -p 127.0.0.1:9901:9901 -p 10000:10000 -p 127.0.0.1:10001:10001 -p127.0.0.1:10002:10002 -v $(pwd)/config.yaml:/config.yaml docker.io/envoyproxy/envoy:v1.37-latest --log-level info -c config.yaml
```

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

## The Really Unfair NetCat Python Sleep Server

```sh
while true; do python3 -c 'fizzbuzz=",".join("Fizz"*(i%3==0) + "Buzz"*(i%5==0) or str(i) for i in range(1,101));print(f"HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\ncontent-length: {len("FizzBuzz:\n")+len(fizzbuzz)+1}\r\nConnection: close\r\n\r\nFizzBuzz:\n"+fizzbuzz);import time; time.sleep(65/1000)' | nc -l 10000; done
```

```sh
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

## The Really Unfair NetCat Python Server

Without `sleep`.

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

## NetCat Static HTML

```sh
while true; do echo -e "HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\ncontent-length: 423\r\nConnection: close\r\n\r\nFizzBuzz:\n1,2,Fizz,4,Buzz,Fizz,7,8,Fizz,Buzz,11,Fizz,13,14,FizzBuzz,16,17,Fizz,19,Buzz,Fizz,22,23,Fizz,Buzz,26,Fizz,28,29,FizzBuzz,31,32,Fizz,34,Buzz,Fizz,37,38,Fizz,Buzz,41,Fizz,43,44,FizzBuzz,46,47,Fizz,49,Buzz,Fizz,52,53,Fizz,Buzz,56,Fizz,58,59,FizzBuzz,61,62,Fizz,64,Buzz,Fizz,67,68,Fizz,Buzz,71,Fizz,73,74,FizzBuzz,76,77,Fizz,79,Buzz,Fizz,82,83,Fizz,Buzz,86,Fizz,88,89,FizzBuzz,91,92,Fizz,94,Buzz,Fizz,97,98,Fizz,Buzz" | nc -l 10000; done
```

```sh
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

## Actual Python Webserver

also, `python3 -m http.server 10000` which serves index.html.

```sh
$ wrk --threads 1 http://127.0.0.1:10000/
Running 10s test @ http://127.0.0.1:10000/
  1 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     4.92ms    1.45ms  27.95ms   75.92%
    Req/Sec     1.39k    38.04     1.44k    95.00%
  13872 requests in 10.00s, 8.06MB read
Requests/sec:   1387.07
Transfer/sec:    824.93KB
```

## nginx

Also, `podman run --rm -it -p 10000:80 -v $(pwd)/site:/usr/share/nginx/html:ro docker.io/nginx:alpine` is fast!

```sh
$ wrk --threads 1 http://127.0.0.1:10000/
Running 10s test @ http://127.0.0.1:10000/
  1 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     1.07ms  775.20us  21.30ms   93.51%
    Req/Sec     9.51k     1.12k   12.15k    71.00%
  94589 requests in 10.00s, 59.63MB read
Requests/sec:   9458.41
Transfer/sec:      5.96MB
```

same with actual 65ms delay in **both** directions
`sudo tc qdisc add dev lo root netem delay 65ms`

```sh
$ wrk --threads 1 http://127.0.0.1:10000/
Running 10s test @ http://127.0.0.1:10000/
  1 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   131.20ms  306.74us 132.89ms   76.80%
    Req/Sec    83.67     24.08   101.00     68.00%
  750 requests in 10.04s, 484.13KB read
Requests/sec:     74.69
Transfer/sec:     48.21KB
```
