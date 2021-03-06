I thought this exercise was pretty difficult.  At some point, I abandoned the verbatim description in the exercise, and I just tried to come up with any solution.  After I figured out a solution, I added the user interface: `start()`, `stop()`.

```erlang
-module(e4).
%%-compile(export_all).
-export([func_init/0, func/0, monitor_init/0, my_monitor/0]).
-export([start/0, stop/0]).
-include_lib("eunit/include/eunit.hrl").

func_init() ->
    register(?MODULE, Pid = spawn(e4, func, []) ),
    Pid.

func() ->
    receive
        stop -> ok
    after 1000 ->
            io:format("I'm still running.~n"),
            func()
    end.

%%-------------

monitor_init() ->
    register(monitor, spawn(?MODULE, my_monitor, [])).
    
my_monitor() ->
    Func = func_init(),
    io:format("Func is running in process: ~w~n", [Func]),
    Ref = monitor(process, Func), 
 
    receive
        {'DOWN', Ref, process, Func, Why} ->
            io:format("~w went down: ~w~nrestarting...~n", [Func, Why]),
            my_monitor();
        {request, stop} ->
            Func ! stop
    end.

%%-----------

start() ->
    monitor_init().

stop() ->
    monitor ! {request, stop},
    shutdown.
    
monitor_test() ->
    start(),

    timer:sleep(3200),
    exit(whereis(e4), my_stop),
    timer:sleep(4200),
    exit(whereis(e4), kill),
    timer:sleep(2200),

    stop().
```

In the shell:

```
$ erl
Erlang/OTP 19 [erts-8.2] [source] [64-bit] [smp:4:4] [async-threads:10] [hipe] [kernel-poll:false]
Eshell V8.2  (abort with ^G)

1> c(e4).
{ok,e4}

2> e4:monitor_test().
Func is running in process: <0.44.0>
I'm still running.
I'm still running.
I'm still running.
<0.44.0> went down: my_stop
restarting...
Func is running in process: <0.45.0>
I'm still running.
I'm still running.
I'm still running.
I'm still running.
<0.45.0> went down: killed
restarting...
Func is running in process: <0.46.0>
I'm still running.
I'm still running.
shutdown

3> 




```



        
    
                  
                  
                  
                
