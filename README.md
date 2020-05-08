# auth-request

auth-request allows you to add access control to your HTTP services based
on a subrequest to a configured haproxy backend. The workings of this Lua
script are loosely based on the [ngx_http_auth_request_module][1] module
for nginx.

## Requirements

### Required

- haproxy 1.8.4+
- `USE_LUA=1` set at compile time.
- LuaSocket with commit [0b03eec16b](https://github.com/diegonehab/luasocket/commit/0b03eec16be0b3a5efe71bcb8887719d1ea87d60) (that is: newer than 2014-11-10) in your Lua library path (`LUA_PATH`).
  - `lua-socket` from Debian Stretch works.
  - `lua-socket` from Ubuntu Xenial works.
  - `lua-socket` from Ubuntu Bionic works.
  - `lua5.3-socket` from Alpine 3.8 works.
  - `luasocket` from luarocks *does not* work.
  - `lua-socket` v3.0.0.17.rc1 from EPEL *does not* work.
  - `lua-socket` from Fedora 28 *does not* work.

## Set-Up

1. Load this Lua script in the `global` section of your `haproxy.cfg`:
```
global
	# *snip*
	lua-load /usr/share/haproxy/auth-request.lua
```
2. Define a backend that is used for the subrequests:
```
backend auth_request
	mode http
	server auth_request 127.0.0.1:8080 check
```
3. Execute the subrequest in your frontend (as early as possible):
```
frontend http
	mode http
	bind :::80 v4v6
	# *snip*

	#                             Backend name     Path to request
	http-request lua.auth-request auth_request     /is-allowed
```
4. Act on the results:
```
frontend http
	# *snip*
	
	http-request deny if ! { var(txn.auth_response_successful) -m bool }
```

## The Details

The Lua script will make a HTTP request to the *first* server in the given
backend that is either marked as `UP` or that does not have checks enabled.
This allows for basic health checking of the auth-request backend. If you
need more complex processing of the request forward the auth-request to a
separate haproxy *frontend* that performs the required modifications to the
request and response.

The requested URL is the one given in the second parameter.

Any request headers will be forwarded as-is to the auth-request backend.

The Lua script will define the `txn.auth_response_successful` variable as
true iff the subrequest returns an HTTP status code of `2xx`. The status code
of the subrequest will be returned in `txn.auth_response_code`. If the
subrequest does not return a valid HTTP response the status code is set
to `500 Internal Server Error`.

Iff the auth backend returns a status code indicating a redirect (301, 302, 303,
307, or 308) the `txn.auth_response_location` variable will be filled with the
contents of the `location` response header.

The auth backend's response headers are returned as individual variables
starting with `txn.auth_request_header.*`. All of the returned header names
are normalized by stripping leading dots, replacing all dots, dashes, and
spaces with underscores, removing all characters other than alphanumerics and
underscores, then lower-casing the resulting header name. This name is then
used to create the variable name.

For example, the `X-Example-Foo Bar!` header will be converted to
`x_example_foo_bar`, and it will be made available in the
`txn.auth_request_header.x_example_foo_bar` variable.

To avoid ambiguity, a comma-separated list of headers is returned in the
`txn.auth_request_header_names` variable.

## Known limitations

- The Lua script only supports basic health checking, without redispatching
  or load balancing of any kind.
- The backend must not be using TLS.

[1]: http://nginx.org/en/docs/http/ngx_http_auth_request_module.html
