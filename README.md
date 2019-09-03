# Fencer

Fencer is a port of <https://github.com/lyft/ratelimit> into Haskell. The
key difference is that Fencer does not use Redis. The API, rule matching
logic, etc, are compatible with `lyft/ratelimit` as far as possible.

## Roadmap

Done:

* In-memory counting
* Rule matching
* `lyft/ratelimit`-compatible gRPC interface
* Configuration loading and reloading

Left to do:

* Garbage collect old counters
* API for setting configuration
* Doctests
* Tests
* Benchmarks

## Building


Install [Nix](https://nixos.org/nix/). On macOS and Linux, this can be done
with:

```
$ curl https://nixos.org/nix/install | sh
```

Then run:

```
$ nix-build
```

## Developing

Install Nix as per instructions in the "Building" secion. Enter the Nix
shell and build the project with `cabal`:

```
$ nix-shell
$ cabal v2-build
```

You can use [`nix-cabal`](https://github.com/monadfix/nix-cabal) as a
`cabal` replacement to get seamless nix-shell support in your editor/IDE.

### Regenerating API from Protobuf

`lib/Fencer/Proto.hs` has been generated from a proto3 file used in
`lyft/ratelimit`. This file, with minor modifications, is vendored in
`proto/rls.proto`. To regenerate the Haskell code from the proto3 file, use
the following command:

```
# compile-proto-file is weird and you can't get it to generate a module with
# a normal name, so we have to resort to sed shenanigans
(cd proto; compile-proto-file --proto rls.proto --out ../lib/Fencer/) \
  && mv lib/Fencer/Rls.hs lib/Fencer/Proto.hs \
  && sed -i 's/module Rls/module Fencer.Proto/' lib/Fencer/Proto.hs \
  && sed -i 's/Rls\./Fencer\.Proto\./g' lib/Fencer/Proto.hs
```

## Design

### Intro

Fencer is a rate limiter, but it does not actually proxy connections,
manager firewall rules, etc. The only thing Fencer does is that when it gets
told "hey, I made a request to XYZ", it either says "alright" or "actually
you shouldn’t have made this request because you are over the limit".

### Rate limiting models

There are two models for rate limiting – the "bucket" one and the "rolling
window" one. In the bucket model you have a single counter which is as
granular as the rate-limiting unit – say, if the limit is "N requests per
hour" then the counter will get incremented for all requests for
X:00:00–X:59:59. If you have spent your limit at the beginning of the hour,
you don't get any credit until the next hour starts and the counter resets.

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
