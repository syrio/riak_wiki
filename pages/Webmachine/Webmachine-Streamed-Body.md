Webmachine allows the resource developer to handle request and
response bodies as either whole units (`binary` or `iolist`) or as
many hunks of data.

The body-handling functions are:

* `wrq:req_body/1`
* `wrq:stream_req_body/2`
* `wrq:set_resp_body/2`

The last of these, `wrq:set_resp_body/2`, is also called implicitly
with the return value of any content-producing function such as
`to_html/2`.

The first of these (`req_body`) is the simplest. It will provide the
whole incoming request body as a binary. (Unless the body is too
large, as set by `wrq:set_max_recv_body/2` or defaulting to 50M). For
the majority of resources, this is the easiest way to handle the
incoming request body.

If a resource wants to handle the incoming request body a hunk at a
time, it may call `wrq:stream_req_body/2` instead. Instead of a
binary, this produces a `StreamBody` structure.

<div class="note">
It is an error to call both <code>wrq:req_body/1</code> and
<code>wrq:stream_req_body/2</code> in the execution of a single resource.
</div>

A `StreamBody` is a pair of the form `{Data,Next}` where `Data` is a
binary and `Next` is either the atom `done` signifying the end of the
body or else a 0-arity function that, when called, will produce the
"next" `StreamBody` structure.

The integer parameter to `wrq:stream_req_body/2` indicates the maximum
size in bytes of any hunk from the resulting `StreamBody`.

When a resource provides a body to be sent in the response, it should
use `wrq:set_resp_body/2`. The parameter to this function may be
either an `iolist`, representing the entire body, or else a pair of
the form `{stream, StreamBody}`.

An example may make the usage of this API clearer. A complete and
working resource module using this API in both directions:

```erlang
-module(mywebdemo_resource).
-export([init/1, allowed_methods/2, process_post/2]).

-include_lib("webmachine/include/webmachine.hrl").

init([]) -> {ok, undefined}.

allowed_methods(ReqData, State) -> {['POST'], ReqData, State}.

process_post(ReqData, State) ->
    Body = get_streamed_body(wrq:stream_req_body(ReqData, 3), []),
    {true, wrq:set_resp_body({stream, send_streamed_body(Body,4)},ReqData), State}.

send_streamed_body(Body, Max) ->
    HunkLen=8*Max,
    case Body of
        <> ->
            io:format("SENT ~p~n",[<>]),
            {<>, fun() -> send_streamed_body(Rest,Max) end};
        _ ->
            io:format("SENT ~p~n",[Body]),
            {Body, done}
    end.

get_streamed_body({Hunk,done},Acc) ->
    io:format("RECEIVED ~p~n",[Hunk]),
    iolist_to_binary(lists:reverse([Hunk|Acc]));
get_streamed_body({Hunk,Next},Acc) ->
    io:format("RECEIVED ~p~n",[Hunk]),
    get_streamed_body(Next(),[Hunk|Acc]).
```

If you use this resource in place of the file
`/tmp/mywebdemo/src/mywebdemo_resource.erl` in the
[[quickstart|Webmachine Quickstart]] setup, you should then be able to
issue `curl -d '1234567890' http://127.0.0.1:8000/` on the command
line and the `io:format` calls will show you what is going on.

Obviously, a realistic resource wouldn't use this API just to collect
the whole binary into memory or break one up that is already present
- you'd use `wrq:req_body` and put a simple `iolist` into
`wrq:set_resp_body` instead. Also, the choices of 3 and 4 bytes as
hunk size are far from optimal for most reasonable uses. This resource
is intended only as a demonstration of the API, not as a real-world
use of streaming request/response bodies.
