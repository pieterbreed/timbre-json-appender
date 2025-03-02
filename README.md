# timbre-json-appender

[![CircleCI](https://circleci.com/gh/viesti/timbre-json-appender.svg?style=svg)](https://circleci.com/gh/viesti/timbre-json-appender) [![Clojars Project](https://img.shields.io/clojars/v/viesti/timbre-json-appender.svg)](https://clojars.org/viesti/timbre-json-appender)

A structured log appender for [Timbre](https://github.com/ptaoussanis/timbre) using [jsonista](https://github.com/metosin/jsonista).

Makes extracting data from logs easier in for example [AWS CloudWatch
Logs](https://aws.amazon.com/about-aws/whats-new/2015/01/20/amazon-cloudwatch-logs-json-log-format-support/) and [GCP Stackdriver Logging](https://cloud.google.com/logging/).

A Timbre log invocation maps to JSON messages the following way:

```clojure
(timbre/error (IllegalStateException. "Not logged in") "Action failure" :user-id 1)
=>
{"timestamp": "2019-07-03T10:00:08Z", # Always included
 "level": "error",                    # ditto
 "thread": "nRepl-session-...",       # ditto
 "hostname": "localhost",             # ditto
 "msg": "Action failure",             # An (optional) first argument
 "user-id": 1,                        # Keyword style arguments
 "err": {"via":[{"type":"...,         # When exception is logged, a Throwable->map presentation of the exception
 "ns": "user",                        # Included when exception is logged
 "file": "...",                       # ditto
 "line": "..."}                       # ditto
```

Note that in version `0.1.1`, `:inline-args?` became the default style, and previously arguments were placed under `:args` key. This output style is still available with `:inline-args? false`:

```
...
"args": {"user-id": 1},
...
```

## Usage

```clojure
user> (require '[timbre-json-appender.core :as tas])
user> (tas/install) ;; Set json-appender as sole Timbre appender
user> (require '[taoensso.timbre :as timbre])

user> (timbre/info "Hello" :user-id 1 :profile {:role :tester}) ;; keyword style args (supports varg pairs with an optional leading message)
{"timestamp":"2019-07-03T10:00:08Z","level":"info","thread":"nRepl-session-97b9389e-a563-4f0d-8b8a-f58050297092","msg":"Hello","args":{"user-id":1,"profile":{"role":"tester"}}}

user> (timbre/info "Hello" {:user-id 1 :profile {:role :tester}}) ;; map style args (supports a single map with an optional leading message)
{"timestamp":"2019-07-03T10:00:08Z","level":"info","thread":"nRepl-session-97b9389e-a563-4f0d-8b8a-f58050297092","msg":"Hello","args":{"user-id":1,"profile":{"role":"tester"}}}

user> (tas/install {:pretty true}) ;; For repl only

user> (timbre/info "Hello" :user-id 1 :profile {:role :tester})
{
  "timestamp" : "2019-07-03T10:23:38Z",
  "level" : "info",
  "hostname": "localhost",
  "thread" : "nRepl-session-97b9389e-a563-4f0d-8b8a-f58050297092",
  "msg" : "Hello",
  "args" : {
    "user-id" : 1,
    "profile" : {
      "role" : "tester"
    }
  }
}
```

Note the expected format:

-   An (optional) leading message is used as the as `msg` key
-   Any subsequent keyword-pair style args (e.g. `:arg-1 1 :arg-2 "value"`) are added to the `args` map
-   If no message is provided, keyword-pair style args will be taken as `args`
-   If a message and a map is provided, the message will be used as `msg` and the hash-map will be taken as `args`
-   If only a hash-map is provided, it will be taken as `args`

Exceptions are included in `err` field via `Throwable->map` and contain `ns`, `file` and `line` fields:

```clojure
user> (tas/install)
user> (timbre/info (IllegalStateException. "Not logged in") "Hello" :user-id 1 :profile {:role :tester})
(timbre/info (IllegalStateException. "Not logged in") "Hello" :user-id 1 :profile {:role :tester})
{"args":{"user-id":1,"profile":{"role":"tester"}},"ns":"user","file":"*cider-repl home/timbre-json-appender:localhost:49943(clj)*","line":523,"err":{"via":[{"type":"java.lang.IllegalStateException","message":"Not logged in","at":["user$eval11384$fn__11385","invoke","NO_SOURCE_FILE",523]}],"trace":[["user$eval11384$fn__11385","invoke","NO_SOURCE_FILE",523],["clojure.lang.Delay","deref","Delay.java",42],["clojure.core$deref","invokeStatic","core.clj",2320],["clojure.core$deref","invoke","core.clj",2306]
```

Data that isn't serializable is omitted, to not prevent logging:

```clojure
user> (tas/install {:pretty true}) ;; For repl only
user> (timbre/info "Hello" :o (Object.))
{
  "timestamp" : "2019-07-03T10:26:38Z",
  "level" : "info",
  "thread" : "nRepl-session-97b9389e-a563-4f0d-8b8a-f58050297092",
  "msg" : "Hello",
  "args" : {
    "o" : { }
  }
}

As a last resort, default println appender is used, if JSON serialization fails.

```

Arguments can also be placed inline, instead of being put behind `:args` key.

```shell
user> (tas/install {:inline-args? true}) ;; Note that :inline-args? defaults to true in tas/install
user> (timbre/info "Hello" :role :admin)
{"timestamp":"2020-09-18T20:26:59Z","level":"info","thread":"nREPL-session-0ac148ff-e0c2-4578-ac64-e5411de14d1f","msg":"Hello","role":"admin"}
nil
```

If you use Timbre's [`with-context`](http://ptaoussanis.github.io/timbre/taoensso.timbre.html#var-with-context),
it will be added to your output automatically (and respects inline-args settings too)

```shell
user=> (tas/install)
user=> (timbre/with-context {:important-context "goes-here" :and :here} (timbre/info "test"))
{"timestamp":"2020-11-03T11:24:45Z","level":"info","thread":"main","msg":"test","important-context":"goes-here","and":"here"}
user=> (tas/install {:inline-args? false})
user=> (timbre/with-context {:important-context "goes-here" :and :here} (timbre/info "test"))
{"timestamp":"2020-11-03T11:25:14Z","level":"info","thread":"main","msg":"test","args":{"important-context":"goes-here","and":"here"}}

```

If you need to emit the log-level to a key other than `level`, you can supply the `level-key` arg

```shell
user=> (tas/install)
user=> (timbre/info "test")
{"timestamp":"2020-11-07T00:28:36Z","level":"info","thread":"main","msg":"test"}
user=> (tas/install {:level-key :severity})
user=> (timbre/info "test")
{"timestamp":"2020-11-07T00:28:50Z","severity":"info","thread":"main","msg":"test"}
```

If you need to emit the message to a key other than `msg`, you can supply the `msg-key` arg.

```shell
user=> (tas/install)
user=> (timbre/info "test")
{"timestamp":"2020-11-07T00:28:36Z","level":"info","thread":"main","msg":"test"}
user=> (tas/install {:msg-key :message})
user=> (timbre/info "test")
{"timestamp":"2020-11-07T00:28:50Z","severity":"info","thread":"main","message":"test"}
```

Map can be passed as an argument, to merge data onto the log output map. This avoids the need to use keyword style to get data onto the top level output map:

```shell
user=> (tas/install)
user=> (timbre/info "Hello" {:user-id 1})
{"timestamp":"2021-03-26T15:05:16Z","level":"info","thread":"main","msg":"Hello","user-id":1}
```

If you wish to change the default fields: `:hostname :thread :ns :file :line` which are logged a function: `:should-log-field-fn` with the signature `(field-name, timbre-data) -> boolean` can be provided which should return a boolean indicating whether to log the field.
By default `tas/default-should-log-field-fn` is used. This only logs `:hostname` and `:thread` unless an error occurs, in which case `:ns`, `:file` and `:line` are also output.

# Changelog

2021-08-28 (0.2.3)

- Allow to override msg-key (eg to support logz.io requiring a field named "message")

2021-07-31 (0.2.2)

- Make printing atomic

2021-04-26 (0.2.1)

- Allow to filter default fields via should-log-field-fn. Add hostname field.

2021-03-26 (0.2.0)

-   Support maps as argument

2020-11-07 (0.1.3)

-   Support to change the level key from level to (eg severity to support GCP Logging)

2020-11-03 (0.1.2)

-   Support timbre/with-context

2020-11-02 (0.1.1)

-   Support inlining arguments

2020-08-13

-   Use taoensso.timbre/println-appender as fallback if JSON serialization fails

2019-08-11

-   Create object mapper only once (improves performance)
-   Support format string style log formatting

2019-07-03

-   Initial release
