# VerneMQ Webhooks

[![Build Status](https://travis-ci.org/erlio/vmq_webhooks.svg?branch=master)](https://travis-ci.org/erlio/vmq_webhooks)

VerneMQ Webhooks enables you to develop VerneMQ plugins in your favorite
programming language. Java, Python, NodeJS, Go, or whatever you current weapon
of choice is doesn't matter. All you need to do is to implement our web-hook
interface and off you go!

## When to use VerneMQ Webhooks

Webhooks has the advantage that you can implement the webhooks in any language
you can use to implement an http interface.

Of course this advantage comes with some trade-offs that one should be aware
of.

Firstly, it should be clear that webhooks are not as efficient a way to
implement plugins as pure erlang plugins.

Secondly, if the webhook endpoints are not available due to network issues or
similar, clients won't be able to connect, publish or subscribe and the effects
of notification webhooks like `on_register`, `on_subscribe`, etc will be lost.

Also, please note that currently, the hooks are invoked with no form of
transport security. Plain HTTP, no authentication, no TLS. A solution for this
is to set up encrypted tunnels through which the VerneMQ Webhooks plugin can
reach the endpoints.

For the above reasons, we recommend that in your production environment you
deploy your endpoints on the same machines where you are deploying VerneMQ and
configure the endpoints to be reached via `localhost`.

## Changelog

See [changelog.md](./changelog.md) for changes.

## Usage

Building the plugin:

    $ ./rebar3 compile

Enabling the plugin:

    $ vmq-admin plugin enable --name=vmq_webhooks --path=/Abs/Path/To/vmq_webhooks/_build/default/

Registering a hook with an endpoint:

    $ vmq-admin webhooks register hook=auth_on_register endpoint="http://localhost"

Deregistering an endpoint:

    $ vmq-admin webhooks deregister hook=auth_on_register endpoint="http://localhost"

The payload is by default base64 encoded, to disable this add the
`--base64payload=false` flag when registering the hook.

## Persisting hooks across VerneMQ restarts

Webhooks added with `vmq-plugin` command line tool are not persisted across
VerneMQ restarts. To make hooks persistent they can be added to the
`priv/vmq_webhooks.conf` file. It contains an example and is hopefully
self-explanatory.

## Webhooks

All webhooks are called with method `POST`. All hooks need to be answered with
the HTTP code `200` to be considered successful. Any hook called that does not
return the `200` code will be logged as an error as will any hook with an
unparseable payload.

All hooks are called with the header `vernemq-hook` which contains the name of
the hook in question.

For detailed information about the hooks and when they are called, see the
[Plugin Development Guide](http://vernemq.com/docs/plugindevelopment/) and the
relevant subsections.

### auth_on_register

Header: ```vernemq-hook: auth_on_register```

Webhook example payload:

```json
{
    "peer_addr": "127.0.0.1",
    "peer_port": 8888,
    "client_id": "clientid",
    "username": "username",
    "password": "password",
    "mountpoint": "",
    "clean_session": false
}
```

Example response:

```json
{
    "result": "ok",
    "modifiers": {
        "mountpoint": "newmountpoint"
        "client_id": "clientid",
        "reg_view": "reg_view",
        "clean_session": false,
        "max_message_size": 65535,
        "max_message_rate": 10000,
        "max_inflight_messages": 100,
        "retry_interval": 100,
        "upgrade_qos": false,
        "trade_consistency": false
    }
}
```

Other result values:

```json
"result": "next"
```

```json
"result": { "error": "some error message" }
```

### auth_on_subscribe

Header: ```vernemq-hook: auth_on_subscribe```

Webhook example payload:

```json
{
    "client_id": "clientid",
    "mountpoint": "",
    "username": "username",
    "topics":
        [{"topic": "a/b",
          "qos": 1},
         {"topic": "c/d",
          "qos": 2}]
}
```

Example response:

```json
{
    "result": "ok",
    "topics":
        [{"topic": "rewritten/topic",
          "qos": 0}]
}
```

Other result values:

```json
"result": "next"
```

```json
"result": { "error": "some error message" }
```

### auth_on_publish

Header: ```vernemq-hook: auth_on_publish```

Note, in the example below the payload is not base64 encoded which is not the
default.

Webhook example payload:

```json
{
    "username": "username",
    "client_id": "clientid",
    "mountpoint": "",
    "qos": 1,
    "topic": "a/b",
    "payload": "hello",
    "retain": false
}
```

Example response:

```json
{
    "result": "ok",
    "modifiers": {
        "topic": "rewritten/topic",
        "qos": 2,
        "payload": "rewritten payload",
        "retain": true,
        "reg_view": "reg_view",
        "mountpoint": "newmountpoint"
    }
}
```

Other result values:

```json
"result": "next"
```

```json
"result": { "error": "some error message" }
```

### on_register

Header: ```vernemq-hook: on_register```

Webhook example payload:

```json
{
    "peer_addr": "127.0.0.1",
    "peer_port": 8888,
    "username": "username",
    "mountpoint": "",
    "client_id": "clientid"
}
```

The response of this hook should be empty as it is ignored.

### on_publish

Header: ```vernemq-hook: on_publish```

Note, in the example below the payload is not base64 encoded which is not the
default.

Webhook example payload:

```json
{
    "username": "username",
    "client_id": "clientid",
    "mountpoint": "",
    "qos": 1,
    "topic": "a/b",
    "payload": "hello",
    "retain": false
}
```

The response of this hook should be empty as it is ignored.

### on_subscribe

Header: ```vernemq-hook: on_subscribe```

Webhook example payload:

```json
{
    "client_id": "clientid",
    "mountpoint": "",
    "username": "username",
    "topics":
        [{"topic": "a/b",
          "qos": 1},
         {"topic": "c/d",
          "qos": 2}]
}
```

The response of this hook should be empty as it is ignored.

### on_unsubscribe

Header: ```vernemq-hook: on_unsubscribe```

Webhook example payload:

```json
{
    "username": "username",
    "client_id": "clientid",
    "mountpoint": "",
    "topics":
        ["a/b", "c/d"]
}
```

Example response:

```json
{
    "result": "ok",
    "topics":
        ["rewritten/topic"]
}
```

Other result values:

```json
"result": "next"
```

```json
"result": { "error": "some error message" }
```

### on_deliver

Header: ```vernemq-hook: on_deliver```

Note, in the example below the payload is not base64 encoded which is not the
default.

Webhook example payload:

```json
{
    "username": "username",
    "client_id": "clientid",
    "mountpoint": "",
    "topic": "a/b",
    "payload": "hello"
}
```

Example response:

```json
{
    "result": "ok",
    "modifiers": {
        "topic": "rewritten/topic",
        "payload": "rewritten payload"
    }
}
```

Other result values:

```json
"result": "next"
```

### on_offline_message

Header: ```vernemq-hook: on_offline_message```

Note, in the example below the payload is not base64 encoded which is not the
default.

Webhook example payload:

```json
{
    "client_id": "clientid",
    "mountpoint": "",
    "qos": "1",
    "topic": "sometopic",
    "payload": "payload",
    "retain": false
}
```

The response of this hook should be empty as it is ignored.

### on_client_wakeup

Header: ```vernemq-hook: on_client_wakeup```

Webhook example payload:

```json
{
    "client_id": "clientid",
    "mountpoint": ""
}
```

The response of this hook should be empty as it is ignored.

### on_client_offline

Header: ```vernemq-hook: on_client_offline```

Webhook example payload:

```json
{
    "client_id": "clientid",
    "mountpoint": ""
}
```

The response of this hook should be empty as it is ignored.

### on_client_gone

Header: ```vernemq-hook: on_client_gone```

Webhook example payload:

```json
{
    "client_id": "clientid",
    "mountpoint": ""
}
```

The response of this hook should be empty as it is ignored.

