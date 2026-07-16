# nrepl-client

A small [nREPL](https://nrepl.org) client for [let-go](https://github.com/nooga/let-go),
speaking bencode over TCP. It handles port discovery, the session handshake, and
a streaming eval loop, so a let-go program can drive a running nREPL server
(let-go's own, a JVM `nrepl` server, or babashka) in a few calls.

Extracted from [nreplctl](https://github.com/abogoyavlensky/nreplctl), which is
its first consumer.

## Install

Add it to `lgx.edn` under `:deps`:

```clojure
{:deps {nrepl-client {:git/url "https://github.com/abogoyavlensky/nrepl-client"
                      :git/tag "v0.1.0"}}}
```

> The tagged release is published at the **first release**, which is pending an
> upstream `let-go` release that ships the `net`/`bencode` namespaces. Until
> then, depend on it by `:local/root` (below) or a specific `:git/sha`.

For local development against a checkout, use a path instead (lgx forbids mixing
`:local/root` with `:git/*`):

```clojure
{:deps {nrepl-client {:local/root "../nrepl-client"}}}
```

## Usage

```clojure
(require '[nrepl-client.core :as nrepl])

(let [host "127.0.0.1"
      port (nrepl/resolve-port {:port nil :port-file ".nrepl-port"})
      conn (nrepl/connect! host port)
      session (nrepl/clone! conn host port)]
  (nrepl/eval! conn session "(+ 1 1)"
               {:emit (fn [[kind s]] (println kind s))})   ; => :value 2
  (nrepl/close! conn session))
```

## API

All functions live in `nrepl-client.core`.

| Function | Purpose |
|---|---|
| `(resolve-port {:port <str-or-nil> :port-file <path>})` | Resolve the port: explicit `:port` wins, else read `:port-file` (e.g. `.nrepl-port`). Returns an int or throws `ex-info` with a friendly message. |
| `(connect! host port)` | Open a TCP connection. `net/dial` errors propagate. |
| `(clone! conn host port)` | Handshake: send `clone`, return the new-session id. Throws if the peer isn't a real nREPL server. |
| `(eval! conn session code {:timeout-ms n-or-nil :emit f})` | Evaluate `code`; call `emit` with each `[:out s]` / `[:err s]` / `[:value s]` event as it streams. Returns `{:error? bool :timeout? bool}`. Does **not** close, so one session serves several evals. |
| `(msg-effects msg)` | Pure: an nREPL message map → `{:events [...] :done? bool :error? bool}`. |
| `(close! conn session)` | Best-effort `close` op, then close the socket. A nil session just closes the socket. |

Each request carries a unique message id and the read loops ignore responses
that aren't theirs, so a session stays usable after a timeout (the still-running
timed-out request's late reply is skipped, not misread as the next eval's result).

## Development

> **Temporary — build lg from source.** nrepl-client needs let-go's `net` and
> `bencode` namespaces, which are not yet in a released `lg`. Until a release
> ships, build lg from the
> [`tcp-client`](https://github.com/abogoyavlensky/let-go/tree/tcp-client) branch
> and point lgx at it via `LGX_LG`:
>
> ```bash
> git clone -b tcp-client https://github.com/abogoyavlensky/let-go /tmp/let-go
> (cd /tmp/let-go && go build -o lg .)
> export LGX_LG=/tmp/let-go/lg
> ```
>
> `.mise.toml` still pins `lg 1.11.1`; bump it and drop this note once a release
> ships `net`/`bencode`.

```bash
mise trust && mise install
lgx test    # run the suite
lgx fmt     # format
lgx lint    # lint (clj-kondo)
lgx check   # fmt check + lint + test
```
