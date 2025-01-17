# Fencer

Fencer is a port of <https://github.com/lyft/ratelimit> into Haskell. The
key difference is that Fencer does not use Redis. The API, rule matching
logic, etc, are compatible with `lyft/ratelimit` as far as possible.

Questions? [Open an issue](https://github.com/juspay/fencer/issues) or get
in touch at <opensource@juspay.in>.

## Usage

We publish Docker images of Fencer on DockerHub at [juspayin/fencer][]. You
can either use images tagged with commit hashes, or with `master` for the
latest build.

[juspayin/fencer]: https://hub.docker.com/r/juspayin/fencer

```
docker pull juspayin/fencer:master
```

You will need the following directory structure:

```
.
├── current -> config1    # symlink to ./config1
└── config1
    └── ratelimit
        └── config
            ├── some_rule.yaml
            └── another_rule.yaml
```

Start Fencer:

```
docker run -d \
  -p 8081:8081 \
  -v $(pwd):/srv/runtime_data \
  -e RUNTIME_SUBDIRECTORY=ratelimit \
  juspayin/fencer
```

To modify configuration, create a new directory (e.g. `./config2`) and
update the `current` symlink to point to it. The configuration will be
reloaded automatically. If the configuration in `./config2` is invalid,
Fencer will keep using the existing configuration.

In production it is recommended to set logging level to `Info` instead of
`Debug` – this significantly increases Fencer's throughput:

```
docker run -d \
  -p 8081:8081 \
  -v $(pwd):/srv/runtime_data \
  -e RUNTIME_SUBDIRECTORY=ratelimit \
  -e LOG_LEVEL=Info \
  juspayin/fencer
```

## Building

Install [Nix](https://nixos.org/nix/). On macOS and Linux, this can be done
with:

```
curl https://nixos.org/nix/install | sh
```

Then build a binary from `default.nix`:

```
nix-build
```

## Docker image

A Docker image can be built from `docker.nix`. The Fencer binary is still
built on the host machine and copied into the image, so you should run
`nix-build docker.nix` from inside a `nixos/nix` container if you want to
get a working image.

Enter `nixos/nix`:

```
docker run -it nixos/nix -v $(pwd):/src
```

From inside, run:

```
cd /src
dockerpath=$(nix-build docker.nix)
cp $dockerpath fencer.tar.gz
```

To load the image into host Docker, run:

```
docker load -i fencer.tar.gz
```

To speed up the build, you can fetch prebuilt dependencies (like gRPC) from
[Cachix](https://cachix.org):

```
nix-env -iA cachix -f https://cachix.org/api/v1/install
cachix use fencer
```

## Environment variables

There are environment variables that can be optionally set, which can
have an effect on Fencer execution.

For logging there is one environment variable:

- `LOG_LEVEL` - The logging level. It can be one of the following:
  trace, debug, info, warn, error and fatal. The default value is
  debug.

Fencer-specific environment variables are:

- `RUNTIME_ROOT` - Symlink to a directory containing the settings
  subdirectory. The default value is `/srv/runtime_data/current`.
- `RUNTIME_SUBDIRECTORY` - The directory with Fencer settings.
- `RUNTIME_IGNOREDOTFILES` - A flag indicating whether to ignore files
  and directories with names starting with a dot (hidden files on
  Linux-based systems and macOS). It can be `True` or `False`. The
  default value is `False`.
- `GRPC_PORT` - The port to run the gRPC server on. Default is 8081.

## Developing

Install Nix as per instructions in the "Building" section. Enter the Nix
shell and build the project with `cabal`:

```
nix-shell
cabal build
```

You can use [`nix-cabal`](https://github.com/monadfix/nix-cabal) as a
`cabal` replacement to get seamless nix-shell support in your editor/IDE.

### Regenerating API from Protobuf

`lib/Fencer/Proto.hs` has been generated from a proto3 file used in
`lyft/ratelimit`. This file, with minor modifications, is vendored in
`proto/rls.proto`. To regenerate the Haskell code from the proto3 file, go
into `nix-shell` and use the following command:

```bash
./scripts/protoGen.sh
```

## Testing

### Go integration tests

To run Go integration tests ported from `lyft/ratelimit`, run in one terminal:

```bash
nix-build

RUNTIME_ROOT=./test_integration_go/config RUNTIME_SUBDIRECTORY=ratelimit result/bin/fencer
```

And in another terminal:

```bash
nix-build test_integration_go

result-bin/bin/test_integration_go
```

## Benchmarking

To run benchmarks, install [ghz][], a gRPC benchmarking tool. A prebuilt
executable can be downloaded from its [GitHub releases page][releases].

[ghz]: https://ghz.sh/
[releases]: https://github.com/bojand/ghz/releases

We also assume that you have `lyft/ratelimit` in your GOPATH (e.g. `~/go/`).
We will use a sample ruleset from lyft/ratelimit to compare the
implementations.

Benchmarking Fencer:

```bash
# In the Fencer repo
LOG_LEVEL=Info \
  RUNTIME_ROOT=~/go/src/github.com/lyft/ratelimit/examples \
  RUNTIME_SUBDIRECTORY=ratelimit \
  result/bin/fencer
```

```bash
# In the Fencer repo
ghz --insecure \
  --proto proto/rls.proto \
  --call envoy.service.ratelimit.v2.RateLimitService/ShouldRateLimit \
  --data '{"domain":"mongo_cps","descriptors":[{"entries":[{"key":"database","value":"users"}]}]}' \
  -n 50000 \
  localhost:8081
```

Benchmarking lyft/ratelimit:

```bash
# In the lyft/ratelimit repo
echo "{version: '3', services: {ratelimit: {environment: [LOG_LEVEL=info]}}}" > docker-compose.override.yml

docker-compose up
```

```bash
# In the Fencer repo
ghz --insecure \
  --proto proto/rls.proto \
  --call envoy.service.ratelimit.v2.RateLimitService/ShouldRateLimit \
  --data '{"domain":"mongo_cps","descriptors":[{"entries":[{"key":"database","value":"users"}]}]}' \
  -n 50000 \
  localhost:8081
```

## Design

### Intro

Fencer is a rate limiter, but it does not actually proxy connections,
manager firewall rules, etc. The only thing Fencer does is that when it gets
told "hey, I made a request to XYZ", it either says "alright" or "actually
you shouldn’t have made this request because you are over the limit".

### Rate limiting models

There are two models for rate limiting – the "bucket" one and the
"rolling window" one. In the bucket model you have a single counter
which is as granular as the rate-limiting unit – say, if the limit is
"N requests per hour" then the counter will get incremented for all
requests for X:00:00–X:59:59. If you have spent your limit at the
beginning of the hour, you don't get any credit until the next hour
starts and the counter resets. Every counter is indexed by a time unit
so that if there is an update in the configuration and rules are
reloaded as a result, a change in time limit yields a new reset
counter; if there is an update in the request rate only, an existing
counter is updated.

The "rolling window" model is more interesting. In this model, you are
limited based on how many requests you’ve made in the past hour, which
doesn't have to align with the hour boundary. So you would not be allowed,
for instance, to make N requests at 0:59 and then N more requests at 1:01 –
you are forced to spread the requests more evenly.

The rolling window model is not particularly tricky to implement, but
`lyft/ratelimit` implements the bucket model and Fencer is a replacement for
`lyft/ratelimit` – which is why we have started with the bucket model. We
can add support for the rolling window model later.

### Configuration

Let's say the config looks like this:

```yaml
domain: messaging
descriptors:
  # Only allow 5 marketing messages a day
  - key: message_type
    value: marketing
    descriptors:
      - key: to_number
        rate_limit: {unit: day, requests_per_unit: 5}

  # Only allow 100 messages a day to any unique phone number
  - key: to_number
    rate_limit: {unit: day, requests_per_unit: 100}
```

It defines two rate limiting rules:

```
message_type=marketing to_number=$variable
to_number=$variable
```

The incoming requests will be matched against this list as follows. Let's
say you get a request:

```
("message_type", "marketing"), ("to_number", "2061111111")
```

It will match the first rule and a new counter will be created for this
specific rule and variable. In this case the variable is `2061111111`. The
limit for the counter is 5, and so you won't be allowed to send more than
five marketing messages to number `2061111111`.

If you get a request that looks like `("to_number", "2061111111")`, it will
only match the second rule, not the first one, and create a counter that
allows 100 messages per day.

All of this is described in slightly more detail here: <https://github.com/lyft/ratelimit#example-2>.

Rate limits in the middle of the chain don't count. For instance, if you are
given this rule:

```yaml
- key: message_type
  value: marketing
  rate_limit: {unit: day, requests_per_unit: 1000}
  descriptors:
    - key: to_number
```

A request `("message_type", "marketing"), ("to_number", "2061111111")` will
match the rule, but since there is no rate limit set for the leaf of the
tree, this request will not be rate-limited at all. On the other hand, if
the request looked like `("message_type", "marketing")`, it would be limited
at 1000 requests per day.

At least, this is according to our understanding of the logic in the Go
code. Ideally we should test this against `lyft/ratelimit` itself, which is
a pending task.

Keep in mind that Fencer can load configuration from any YAML files
and not just files with the `.yml` extension. For example, Fencer will
load configuration from a YAML file named `limit` so long as the file
is located in the runtime directory given with the
`RUNTIME_SUBDIRECTORY` environment variable. Furthermore, it will load
configuration from a runtime subdirectory recursively, thereby
traversing the whole subdirectory.

## Differences to Lyft's rate limit service

Fencer differs in several ways compared to [Lyft's rate limit
service](https://github.com/lyft/ratelimit/):

* Fencer does not use Redis.

* Fencer is implemented in Haskell, while `lyft/ratelimit` is
  implemented in Go.

* In `lyft/ratelimit`, the `limitRemaining` key is left out altogether
  if the remaining limit is zero. For all non-zero values it is given
  in the usual key-value notation, e.g., `"limitRemaining": 4`. Fencer
  always returns the key, including when the value is zero:
  `"limitRemaining": 0`.

* Due to an implementation detail, `lyft/ratelimit` can resolve rate limit
  requests containing underscores incorrectly. For instance, given the
  following rule:

  ```yaml
  domain: mongo_cps
  descriptors:
    - key: database
      rate_limit:
        unit: second
        requests_per_unit: 500
  ```

  `lyft/ratelimit` will apply it to the following descriptor, even though it
  should not:

  ```
  domain: mongo
  descriptor: ("cps_database", "dbname")
  ```

  Fencer does not suffer from this misfeature.

## Contributing

Please see our [Contribution Guidelines](CONTRIBUTING.md).
