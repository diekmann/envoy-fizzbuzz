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
1,2,Fizz,4,Buzz,Fizz,7,8,Fizz,Buzz,11,Fizz,13,14,FizzBuzz,16,17,Fizz,19,Buzz,Fizz,22,23,Fizz,Buzz,26,Fizz,28,29,FizzBuzz,31,32,Fizz,34,Buzz,Fizz,37,38,Fizz,Buzz,41,Fizz,43,44,FizzBuzz,46,47,Fizz,49,Buzz,Fizz,52,53,Fizz,Buzz,56,Fizz,58,59,FizzBuzz,61,62,Fizz,64,Buzz,Fizz,67,68,Fizz,Buzz,71,Fizz,73,74,FizzBuzz,76,77,Fizz,79,Buzz,Fizz,82,83,Fizz,Buzz,86,Fizz,88,89,FizzBuzz,91,92,Fizz,94,Buzz,Fizz,97,98,Fizz,Buzz
```

Or in the browser directly:

![FizzBuzz shown in the Firefox browser](img/fizzbuzz_firefox.png)

## Architecture

![Architecture of the config.yaml](img/architecture.png)