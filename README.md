

# hackney - HTTP client library in Erlang #

Copyright (c) 2012-2014 Benoît Chesneau.

__Version:__ 0.14.1

# hackney

**hackney** is an HTTP client library for Erlang.

Main features:

- no message passing (except for asynchronous responses): response is
  directly streamed to the current process and state is kept in a `#client{}` record.
- binary streams
- SSL support
- Keepalive handling
- basic authentication
- stream the response and the requests
- fetch a response asynchronously
- multipart support (streamed or not)
- chunked encoding support
- Can send files using the sendfile API
- Chunked encoding support
- Optional socket pool
- REST syntax: `hackney:Method(URL)` (where a method can be get, post, put, delete, ...)

Hackney use [hackney_lib](http://github.com/benoitc/hackney_lib) to
handle any web protocols. If you want to manipulate headers, cookies,
multipart or any other thing related to web protocols this is the place
to go.

> Note: This is a work in progress, see the
[TODO](http://github.com/benoitc/hackney/blob/master/TODO.md) for more
informations on what still need to be done.

#### Useful modules are:

- [`hackney`](http://github.com/benoitc/hackney/blob/master/doc/hackney.md): main module. It contains all HTTP client functions.
- [`hackney_http`](http://github.com/benoitc/hackney/blob/master/doc/hackney_http.md): HTTP parser in pure Erlang. This parser is able
to parse HTTP responses and requests in a streaming fashion. If not set
it will be autodetect if it's a request or a response if needed.

- [`hackney_headers`](http://github.com/benoitc/hackney/blob/master/doc/hackney_headers.md) Module to manipulate HTTP headers
- [`hackney_cookie`](http://github.com/benoitc/hackney/blob/master/doc/hackney_cookie.md): Module to manipulate cookies.
- [`hackney_multipart`](http://github.com/benoitc/hackney/blob/master/doc/hackney_multipart.md): Module to encode/decode multipart.
- [`hackney_url`](http://github.com/benoitc/hackney/blob/master/doc/hackney_url.md): Module to parse and create URIs.
- [`hackney_date`](http://github.com/benoitc/hackney/blob/master/doc/hackney_date.md): Module to parse HTTP dates.

Read the [NEWS](https://raw.github.com/benoitc/hackney/master/NEWS.md) file
to get last changelog.

## Installation

Download the sources from our [Github
repository](http://github.com/benoitc/hackney)

To build the application simply run 'make'. This should build .beam, .app
files and documentation.

To run tests run 'make test'.
To generate doc, run 'make doc'.

Or add it to your rebar config

```
{deps, [
    ....
    {hackney, ".*", {git, "git://github.com/benoitc/hackney.git", {branch, "master"}}}
]}.
```

## Basic usage

The basic usage of hackney is:

### Start hackney

hackney is an
[OTP](http://www.erlang.org/doc/design_principles/users_guide.html)
application. You have to start it first before using any of the functions.
The hackney application will start the default socket pool for you.

To start in the console run:

```
$ erl -pa ebin -pa deps/*/ebin
1>> hackney:start().
ok
```

It will start hackney and all of the application it depends on:

```
application:start(crypto),
application:start(public_key),
application:start(ssl),
application:start(hackney).
```

Or add hackney to the applications property of your .app in a release

### Simple request

Do a simple request that will return a client state:

```
Method = get,
URL = <<"https://friendpaste.com">>,
Headers = [],
Payload = <<>>,
Options = [],
{ok, StatusCode, RespHeaders, ClientRef} = hackney:request(Method, URL,
                                                        Headers, Payload,
                                                        Options).
```

The request method return the tuple `{ok, StatusCode, Headers, ClientRef}`
or `{error, Reason}`. A `ClientRef` is simply a reference to the current
request that you can reuse.

If you prefer the REST syntax, you can also do:

```
hackney:Method(URL, Headers, Payload, Options)
```

where `Method`, can be any HTTP methods in lowercase.

### Read the body

```
{ok, Body} = hackney:body(Client).
```

`hackney:body/1` fetch the body. To fetch it by chunk you can use the
`hackney:stream_body/1` function:

```
read_body(MaxLength, Ref, Acc) when MaxLength > byte_size(Acc) ->
	case stream_body(Ref) of
		{ok, Data} ->
			read_body(MaxLength, Ref, << Acc/binary, Data/binary >>);
		done ->
			{ok, Acc};
		{error, Reason} ->
			{error, Reason}
	end.
```

> Note: you can also fetch a multipart response using the functions
> `hackney:stream_multipart/1` and  `hackney:skip_multipart/1`.

### Reuse a connection

By default all connections are created and closed dynamically by
hackney but sometimes you may want to reuse the same reference for your
connections. It's especially usefull you just want to handle serially a
couple of request.

> A closed connection will automatically be reconnected.

#### To create a connection:

```
Transport = hackney_tcp_transport,
Host = << "https://friendpaste.com" >>,
Port = 443,
Options = [],
{ok, ConnRef} = hackney:connect(Transport, Host, Port, Options)
```

> To create a connection that will use an HTTP proxy use
> `hackney_http_proxy:connect_proxy/5` instead.

#### Make a request

Once you created a connection use the `hackney:send_request/2` function
to make a request:

```
ReqBody = << "{	\"snippet\": \"some snippet\" }" >>,
ReqHeaders = [{<<"Content-Type">>, <<"application/json">>}],
NextPath = <<"/">>,
NextMethod = post,
NextReq = {NextMethod, NextPath, ReqHeaders, ReqBody}
{ok, _, _, ConnRef} = hackney:send_request(ConnRef, NextReq).
{ok, Body1} = hackney:body(ConnRef),
```

Here we are posting a JSON payload to '/' on the friendpaste service to
create a paste. Then we close the client connection.

> If your connection supports keepalive the connection will be simply :

### Send a body

hackney helps you send different payloads by passing different terms as
the request body:

- `{form, PropList}` : To send a form
- `{multipart, Parts}` : to send you body using the multipart API. Parts
  follow this format:
  - `eof`: end the multipart request
  - `{file, Path}`: to stream a file
  - `{file, Path, ExtraHeaders}`: to stream a file
  - `{Name, Content}`: to send a full part
  - `{Name, Content, ExtraHeaders}`: to send a full part
  - `{mp_mixed, Name, MixedBoundary}`: To notify we start a part with a
    a mixed multipart content
  - `{mp_mixed_eof, MixedBoundary}`: To notify we end a part with a a
    mixed multipart content
- `{file, File}` : To send a file
- Bin: To send a binary or an iolist

> Note: to send a chunked request, just add the `Transfer-Encoding: chunked`
> header to your headers. Binary and Iolist bodies will be then sent using
> the chunked encoding.

#### Send the body by yourself

While the default is to directly send the request and fetch the status
and headers, if the body is set as the atom `stream` the request and
send_request function will return {ok, Client}. Then you can use the
function `hackney:send_body/2` to stream the request body and
`hackney:start_response/1` to initialize the response.

> Note: The function `hackney:start_response/1` will only accept
> a Client that is waiting for a response (with a response state
> equal to the atom `waiting`).

Ex:

```
ReqBody = << "{
      \"id\": \"some_paste_id2\",
      \"rev\": \"some_revision_id\",
      \"changeset\": \"changeset in unidiff format\"
}" >>,
ReqHeaders = [{<<"Content-Type">>, <<"application/json">>}],
Path = <<"https://friendpaste.com/">>,
Method = post,
{ok, ClientRef} = hackney:request(Method, Path, ReqHeaders, stream, []),
ok  = hackney:send_body(ClientRef, ReqBody),
{ok, _Status, _Headers, ClientRef} = hackney:start_response(ClientRef),
{ok, Body} = hackney:body(ClientRef),
```

> Note: to send a **multipart** body  in a streaming fashion use the
> `hackney:sen_multipart_body/2` function.

### Get a response asynchronously

Since the 0.6 version, hackney is able to fetch the response
asynchrnously using the `async` option:

```
Url = <<"https://friendpaste.com/_all_languages">>,
Opts = [async],
LoopFun = fun(Loop, Ref) ->
        receive
            {hackney_response, Ref, {status, StatusInt, Reason}} ->
                io:format("got status: ~p with reason ~p~n", [StatusInt,
                                                              Reason]),
                Loop(Loop, Ref);
            {hackney_response, Ref, {headers, Headers}} ->
                io:format("got headers: ~p~n", [Headers]),
                Loop(Loop, Ref);
            {hackney_response, Ref, done} ->
                ok;
            {hackney_response, Ref, Bin} ->
                io:format("got chunk: ~p~n", [Bin]),
                Loop(Loop, Ref);

            Else ->
                io:format("else ~p~n", [Else]),
                ok
        end
    end.

{ok, ClientRef} = hackney:get(Url, [], <<>>, Opts),
LoopFun(LoopFun, ClientRef).
```

> **Note 1**: When `{async, once}` is used the socket will receive only once.
> To receive the other messages use the function `hackney:stream_next/1`.

> **Note 2**:  Asynchronous responses automatically checkout the socket at the end.

> **Note 3**:  At any time you can go back and received your response
> synchronously using the function `hackney:stop_async/1` See the
> example [test_async_once2](https://github.com/benoitc/hackney/blob/master/examples/test_async_once2.erl) for the usage.

> **Note 4**:  When the option `{following_redirect, true}` is passed to
> the request, you will receive the folllowing messages on valid
> redirection:
> - `{redirect, To, Headers}`
> - `{see_other, To, Headers}` for status 303 and POST requests.

> **Note 5**: You can send the messages to another process by using the
> option `{stream_to, Pid}` .

### Use the default pool

To reuse a connection globally in your application you can also use a
socket pool. On startup, hackney launches a pool named default. To use it
do the following:

```
Method = get,
URL = <<"https://friendpaste.com">>,
Headers = [],
Payload = <<>>,
Options = [{pool, default}],
{ok, StatusCode, RespHeaders, ClientRef} = hackney:request(Method, URL, Headers,
                                                        Payload, Options).
```

By adding the tuple `{pool, default}` to the options, hackney will use
the connections stored in that pool.

You can also use different pools in your application which allows
you to maintain a group of connections.

```
PoolName = mypool,
Options = [{timeout, 150000}, {max_connections, 100}],
ok = hackney_pool:start_pool(PoolName, Options),
```

`timeout` is the time we keep the connection alive in the pool,
`max_connections` is the number of connections maintained in the pool. Each
connection in a pool is monitored and closed connections are removed
automatically.

To close a pool do:

```
hackney_pool:stop_pool(PoolName).
```

> Note: Sometimes you want to always use the default pool in your app
> without having to set the client option each time. You can now do this
> by setting the hackney application environment key `use_default_pool`
> to true.

### Use a custom pool handler.

Since the version 0.8 it is now possible to use your own Pool to
maintain the connections in hackney.

A pool handler is a module that handle the `hackney_pool_handler`
behaviour.

See for example the
[hackney_disp](https://github.com/benoitc/hackney_disp) a load-balanced
Pool dispatcher based on dispcount].> Note: for now you can`t force the pool handler / client.

### Automatically follow a redirection

If the option `{follow_redirect, true}` is given to the request, the
client will be able to automatically follow the redirection and
retrieve the body. The maximum number of connections can be set using the
`{max_redirect, Max}` option. Default is 5.

The client will follow redirects on 301, 302 & 307 if the method is
get or head. If another method is used the tuple
`{ok, maybe_redirect, Status, Headers, Client}` will be returned. It
only follow 303 redirects (see other) if the method is a POST.

Last Location is stored in the `location` property of the client state.

ex:

```
Method = get,
URL = "http://friendpaste.com/",
ReqHeaders = [{<<"accept-encoding">>, <<"identity">>}],
ReqBody = <<>>,
Options = [{follow_redirect, true}, {max_redirect, 5}],
{ok, S, H, Ref} = hackney:request(Method, URL, ReqHeaders,
                                     ReqBody, Options),
{ok, Body1} = hackney:body(Ref).
```

### Proxy a connection

#### HTTP Proxy

To use an HTTP tunnel add the option `{proxy, ProxyUrl}` where
`ProxyUrl` can be a simple url or an `{Host, Port}` tuple. If you need
to authenticate set the option `{proxy_auth, {User, Password}}`.

#### SOCKS5 proxy

Hackney supports the connection via a socks5 proxy. To set a socks5
proxy, use the following settings:

- `{proxy, {socks5, ProxyHost, ProxyPort}}`: to set the host and port of
  the proxy to connect.
- `{socks5_user, Username}`: to set the user used to connect to the proxy
- `{socks5_pass, Password}`: to set the password used to connect to the proxy

SSL and TCP connections can be forwarded via a socks5 proxy. hackney is
automatically upgrading to an SSL connection if needed.

## Contribute

For issues, comments or feedback please [create an
issue](http://github.com/benoitc/hackney/issues).

### Notes for developers

If you want to contribute patches or improve the docs, you will need to
build hackney using the `rebar_dev.config`  file. It can also be built
using the **Makefile**:

```
$ make dev ; # compile & get deps
$ make devclean ; # clean all files
```


## Modules ##


<table width="100%" border="0" summary="list of modules">
<tr><td><a href="http://github.com/benoitc/hackney/blob/master/doc/hackney.md" class="module">hackney</a></td></tr>
<tr><td><a href="http://github.com/benoitc/hackney/blob/master/doc/hackney_app.md" class="module">hackney_app</a></td></tr>
<tr><td><a href="http://github.com/benoitc/hackney/blob/master/doc/hackney_bstr.md" class="module">hackney_bstr</a></td></tr>
<tr><td><a href="http://github.com/benoitc/hackney/blob/master/doc/hackney_connect.md" class="module">hackney_connect</a></td></tr>
<tr><td><a href="http://github.com/benoitc/hackney/blob/master/doc/hackney_cookie.md" class="module">hackney_cookie</a></td></tr>
<tr><td><a href="http://github.com/benoitc/hackney/blob/master/doc/hackney_date.md" class="module">hackney_date</a></td></tr>
<tr><td><a href="http://github.com/benoitc/hackney/blob/master/doc/hackney_deps.md" class="module">hackney_deps</a></td></tr>
<tr><td><a href="http://github.com/benoitc/hackney/blob/master/doc/hackney_headers.md" class="module">hackney_headers</a></td></tr>
<tr><td><a href="http://github.com/benoitc/hackney/blob/master/doc/hackney_http.md" class="module">hackney_http</a></td></tr>
<tr><td><a href="http://github.com/benoitc/hackney/blob/master/doc/hackney_http_connect.md" class="module">hackney_http_connect</a></td></tr>
<tr><td><a href="http://github.com/benoitc/hackney/blob/master/doc/hackney_idna.md" class="module">hackney_idna</a></td></tr>
<tr><td><a href="http://github.com/benoitc/hackney/blob/master/doc/hackney_manager.md" class="module">hackney_manager</a></td></tr>
<tr><td><a href="http://github.com/benoitc/hackney/blob/master/doc/hackney_mimetypes.md" class="module">hackney_mimetypes</a></td></tr>
<tr><td><a href="http://github.com/benoitc/hackney/blob/master/doc/hackney_multipart.md" class="module">hackney_multipart</a></td></tr>
<tr><td><a href="http://github.com/benoitc/hackney/blob/master/doc/hackney_pool.md" class="module">hackney_pool</a></td></tr>
<tr><td><a href="http://github.com/benoitc/hackney/blob/master/doc/hackney_pool_handler.md" class="module">hackney_pool_handler</a></td></tr>
<tr><td><a href="http://github.com/benoitc/hackney/blob/master/doc/hackney_request.md" class="module">hackney_request</a></td></tr>
<tr><td><a href="http://github.com/benoitc/hackney/blob/master/doc/hackney_response.md" class="module">hackney_response</a></td></tr>
<tr><td><a href="http://github.com/benoitc/hackney/blob/master/doc/hackney_socks5.md" class="module">hackney_socks5</a></td></tr>
<tr><td><a href="http://github.com/benoitc/hackney/blob/master/doc/hackney_ssl_transport.md" class="module">hackney_ssl_transport</a></td></tr>
<tr><td><a href="http://github.com/benoitc/hackney/blob/master/doc/hackney_stream.md" class="module">hackney_stream</a></td></tr>
<tr><td><a href="http://github.com/benoitc/hackney/blob/master/doc/hackney_sup.md" class="module">hackney_sup</a></td></tr>
<tr><td><a href="http://github.com/benoitc/hackney/blob/master/doc/hackney_tcp_transport.md" class="module">hackney_tcp_transport</a></td></tr>
<tr><td><a href="http://github.com/benoitc/hackney/blob/master/doc/hackney_url.md" class="module">hackney_url</a></td></tr>
<tr><td><a href="http://github.com/benoitc/hackney/blob/master/doc/hackney_util.md" class="module">hackney_util</a></td></tr></table>

