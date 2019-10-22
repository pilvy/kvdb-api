---
title: KVdb API Reference

#language_tabs: # must be one of https://git.io/vQNgJ
#  - shell: cURL
#  - javascript: jQuery

toc_footers:
  - <a href="/docs/" title="KVdb Documentation">Documentation</a>
  - <a href="/" title="KVdb Key Value Database">KVdb Home</a>

includes:
  - errors

search: true
---

# Introduction

Welcome to the <a href="/">KVdb</a> API! You can use our simple REST API to store arbitrary key-value pairs, which can contain any type of data, such as text, numbers, counters, and binary data. Depending on the data type, the API offers additional features to make collecting and accessing data easier.

While all the examples are given in Shell script (using curl on the command line), it's possible to request JSON and URL-encoded output.

# Authentication

> To authenticate with a bucket secret or write key:

```shell
# Provide the key as the username using HTTP Basic authentication
curl https://kvdb.io/BUCKET/KEY
  -u 'mykey:'

# Or in the query string
curl https://kvdb.io/BUCKET/KEY?key=mykey
```

While not all API endpoints require the use of an API key, should you choose to use one, use the key in the username field when sending HTTP Basic authentication credentials. The key can also e specified in the `key` query string parameter.

In any case, use of the bucket secret key provides full access to the bucket, where as the bucket write key provides write access to any keys in the bucket. You SHOULD NOT use either of these in client-side applications.

# Authorization

> To generate an access token valid for 1 hour with read/write permission for the prefix user:123:

```shell
curl https://kvdb.io/BUCKET/tokens/
  -d 'prefix=user:123:&permissions=read,write&ttl=3600'
  -u mykey:
```

> To authorize a request with an access token:

```shell
# Provide the access token in the Authorization header
curl https://kvdb.io/BUCKET/KEY
  -H "Authorization: Bearer MY_TOKEN"

# Or in the query string
curl https://kvdb.io/BUCKET/KEY?access_token=MY_TOKEN
```

For most applications, you do not want to give out the bucket's secret or write keys, since it gives clients access to all keys in the bucket. Instead, you can use the KVdb API to generate access tokens, which grant the holder access to a specific prefixed portion of the key space with an exact set of permissions. By default, access tokens are stateless and have a fixed lifetime.

Key naming schemes are often prefixed with some path containing the application user ID, for example, `user:123:email`, `user:123:profile`, etc. A secure way to grant a user of your application read/write access to only their own set of sub-keys is to generate an access token for the key with their user ID prefix, `user:123:` for instance. Furthermore, the access token can embed only the minimum set of permissions required.

Access tokens offer the ability to securely provide external access to your bucket resources without an intermediate authentication or authorization layer, while still maintining application-level control.


# Rate Limits

The rate limit is 1,000 requests per IP address per hour. Refer to the `X-Ratelimit-Limit`, `X-Ratelimit-Remaining`, and `X-Ratelimit-Reset` response headers to know when you are approaching the limit. Once the rate limit is reached, the server will return a HTTP 429 status code until the rate limit resets.

Paid plans have a much more generous rate limit allowance.

# Buckets

A bucket is a collection of keys. The keys in a bucket share the same access keys, expiration (TTL) setting, and other bucket policy. Depending on your application, you may use one bucket for all your keys, or segregate keys based on users, groups, or other categorization in your application.

You may create bucket-specific access keys:

Key         | Default | Set this to...
----------- | ------- | -----------
secret_key  | None    | Manage bucket policy and other keys
read_key    | None    | Prevent public reads from the bucket
write_key   | None    | Prevent public writes to the bucket
signing_key | None    | Enable access token generation

Set the `signing_key` to a random, yet secure value, as it will be used to generate cryptographically signed access tokens. Changing the `signing_key` will immediately invalidate all previously issued access tokens.

In addition, you may configure the following bucket policies:

Setting     | Default          | Meaning
----------- | ---------------- | -------
default_ttl | 604800 (1 week)  | Keys not updated expire after this time (seconds)

<aside class="notice">
Disabling key expiration (<code>default_ttl=0</code>) or increasing it beyond 1 week requires a paid plan.
</aside>


## Create a Bucket

When creating a bucket, ensure that you set a valid email address in order to be able to manage the bucket in the future and avoid loss of data.

```shell
# Create a new bucket that lets anyone read, write, and list keys
curl -d 'email=user@example.com' https://kvdb.io
```

> The above command returns the bucket id as plain text:

```text
N7cmQg1DwZbADh2Hu3NncF
```

```shell
# Create a new bucket with a secret_key, write_key,
# and set all keys to expire after 1 hour
curl https://kvdb.io \
     -d 'email=user@example.com' \
     -d 'secret_key=supersecret' \
     -d 'write_key=knock' \
     -d 'default_ttl=3600'
```

Our API will generate a bucket id for you to use. Make sure to set a `secret_key` to protect the bucket. It is not possible to add a `secret_key` to a bucket after creation, however, it is possible to change it.

There is no API to list publicly-accessible buckets, so using a bucket without an access key is safe as long as the bucket id isn't made public. We only recommend these kinds of buckets for testing purposes.


## Update Bucket Policy

```shell
# Make bucket read-only
curl https://kvdb.io/N7cmQg1DwZbADh2Hu3NncF \
     -d 'write_key=knock' \
     -u 'supersecret:' \
     -XPATCH
```

If a bucket has a `secret_key` set, you may update its policy (and the `secret_key` itself).


## List Keys

```shell
# List keys
curl https://kvdb.io/N7cmQg1DwZbADh2Hu3NncF/ \
     -u 'supersecret:'

# List keys with the prefix hits_2018
curl 'https://kvdb.io/N7cmQg1DwZbADh2Hu3NncF/?prefix=hits_2018' \
     -u 'supersecret:'

# List keys and their values as JSON
curl 'https://kvdb.io/N7cmQg1DwZbADh2Hu3NncF/?values=true&format=json' \
     -u 'supersecret:'
```

> This would return an array of JSON arrays with key-value pairs:

```shell
[
  ["visitors", 101],
  ["flavor-today", "coconut"]
]
```

Listing keys in a bucket is simple. If the bucket is protected by a `secret_key`, be sure to provide it.

The following query string paramateres can be used to influence the output:

Parameter | Default          | Meaning
--------- | ---------------- | -------
limit     | 10000            | Maximum number of keys to return
skip      | 0                | Number of keys to skip before returning first result
prefix    | _none_           | Only return keys matching the given prefix
values    | false            | Return values
format    | _Accept_ header  | Change the output format (see below)

The HTTP `Accept` header is used to determine the output if the `format` parameter is not specified. If no specific header is sent, `text` is assumed.

The following formats are supported:

Format    | Without Values | With Values          | Value Escaping
--------- | -------------- | -------------------- | ---------------
text      | `key`          | `key=value`          | Newline: `\n`
json      | `["key"]`      | `[["key", value]]`   | None
jsonl     | `"key"`        | `["key", value]`     | None

`jsonl` is newline-delimited JSON, a format easy to process with command-line and other tools expecting one JSON object per line. It must be specifically requested with a `?format=jsonl` query string.


## Delete Bucket

```shell
# Delete bucket
curl https://kvdb.io/N7cmQg1DwZbADh2Hu3NncF/ \
     -XDELETE
     -u 'supersecret:'
```

Use this to delete a bucket and all of its keys. For large buckets, this operation may not complete immediately, even though a successful response is returned by the API.


# Keys

Key names can be of arbitrary composition (even binary data), but are limited to 128 bytes in length. Values can be of any data type but are limited to 16 KB in size. Numeric values are detected automatically and are enforced by the API.


## Set a Key Value

```shell
# Set key
curl https://kvdb.io/N7cmQg1DwZbADh2Hu3NncF/hello \
     -d 'world'

# Set key with a 30 second expiry
curl https://kvdb.io/N7cmQg1DwZbADh2Hu3NncF/hello?ttl=30 \
     -d 'world'
```

The maximum length of a key is 128 bytes.

To override the bucket's default key expiration, use the `ttl` query parameter to specify the TTL for this key.


## Get a Key Value

```shell
# Get value
curl https://kvdb.io/N7cmQg1DwZbADh2Hu3NncF/hello
```

> The response is the value:

```text
world
```


## Update a Key (Counter)

```shell
# Increment counter by 1
curl https://kvdb.io/N7cmQg1DwZbADh2Hu3NncF/visits
     -d '+1'
     -XPATCH
```

> The API responds with the new value of the counter:

```text
1
```

To store counters, the server will detect integer (64-bit signed) and floating point (64-bit) values and store them efficiently, to allow incrementing and decrementing them. Use the `PATCH` HTTP method to operate on a value this way. If the key does not exist, it is created and assumed to be zero before the operation.

To increment the value, use `+`.

To decrement the value, use `-`.

You can also use the `ttl` query parameter to set the TTL for this key.


## Delete a Key

```shell
# Delete key
curl https://kvdb.io/N7cmQg1DwZbADh2Hu3NncF/hello \
     -XDELETE
```

If you need to delete a key before it expires, use this method.


# Scripting

KVdb provides a powerful Lua-based scripting API that lets you build server-side functionality with fast access to keys and values. Each bucket can have one or more scripts, which execute in the context of that bucket and an HTTP request. Furthermore, the keys a script can access can be restricted by the permissions granted by the access token in the HTTP request. With these features, scripts can be used to implement any imaginable level of server-side business logic.

Refer to the <a href="/docs/scripting">Scripting Reference</a> for the full Lua API.

To view and create your scripts, visit https://kvdb.io/BUCKET/scripts/
