# KFlow
> Bullet-proof data stream processing framework

[![Build Status][ci-image]][ci-url]
[![License][license-image]][license-url]

KFlow is an Erlang dataflow DSL built with Kafka in mind. Services
implemented using Kflow have the following properties:

 1. Stateless. Service can be restarted at any moment without risk of
    losing data
 1. Fault-tolerant and self-healing. Any crashes in the service can be
    fixed, and the service will automatically replay the data that
    caused crash, once the fix is deployed. This enables fearless A/B
    testing and canary deployments
 1. Scalable.
 1. Automated parallelization. Each transformation runs in parallel
 1. Automated backpressure.

This is achieved by rather sophisticated offset tracking logic built
in KFlow behaviors, that makes sure that consumer offsets are
committed to Kafka only when an input message is fully processed.

Another feature of KFlow is "virtual partition" that allows to split
Kafka partitions into however many substream that are processed
independently.

## Usage example

KFlow is configured using a special Erlang module named
`kflow_config.erl`. This module must export `pipes/0` function
returning a list of workflows. For example:

```erlang
-module(kflow_config).

-export([pipes/0]).

pipes() -> [example_workflow()].

example_workflow() ->
  %% Define a "pipe" (much like Unix pipe):
  PipeSpec = [ %% Parse messages:
               {map, fun(_Offset, #{key => KafkaKey, value => JSON} ->
                         (jsone:decode(JSON)) #{key => KafkaKey}
                     end}
               %% Create a "virtual partition" for each unique key:
             , {demux, fun(_Offset, #{key := Key}) ->
                           Key
                       end}
               %% Collect messages into chunks of 100 for faster processing:
             , {aggregate, kflow_buffer, #{max_messages => 100}}
               %% Dump chunks into Postgres:
             , {map, kflow_postgres, #{ database => #{ host => "localhost"
                                                     , ...
                                                     }
                                      , table  => "my_table"
                                      , fields => [key, foo, bar, baz]
                                      , keys   => [key]
                                      }}
             ],
  kflow:mk_kafka_workflow(?FUNCTION_NAME, PipeSpec,
                          #{ kafka_topic => <<"example_topic">>
                           , group_id    => <<"example_consumer_group_id">>
                           }).
```

_For more examples and usage, please refer to the [Docs](https://hexdocs.pm/kflow/)._

## Development setup

KFlow requires OTP21 or later and `rebar3` present in the
`PATH`. Build by running

```sh
make
```

## How to contribute

See our guide on [contributing](.github/CONTRIBUTING.md).

## Release History

See our [changelog](CHANGELOG.md).

## License

Copyright © 2020 Klarna Bank AB

For license details, see the [LICENSE](LICENSE) file in the root of this project.


<!-- Markdown link & img dfn's -->
[ci-image]: https://img.shields.io/badge/build-passing-brightgreen?style=flat-square
[license-image]: https://img.shields.io/badge/license-Apache%202-blue?style=flat-square
[license-url]: http://www.apache.org/licenses/LICENSE-2.0
