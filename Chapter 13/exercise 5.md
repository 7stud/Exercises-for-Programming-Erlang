Debugging hell! 

1. I kept a list of the monitored Workers, and each monitored Worker has the following strucuture:
    
        { {Pid, Ref}, Func}
        
    `{Pid, Ref}` is returned by `spawn_monitor(Func)`.

2. Originally, I had `restart_worker()` handle the case where the pid of the killed worker is not found in the list of monitored pids, i.e if `lists:keyfind()` returns false, but I don't think that's possible: it would mean that the monitor received a 'DOWN' message from a process that it wasn't monitoring, so I eliminated that case.

3. At the bottom, there is a version that uses maps in Erlang 19.2.


```erlang
-module(e5).
%%-compile(export_all).
-export([monitor_workers_init/1, monitor_workers/1]).
-export([stop_monitor/1, test/0]).

monitor_workers_init(Funcs) ->
    spawn(?MODULE, monitor_workers, [Funcs]).

monitor_workers(Funcs) ->
    Workers = [ {spawn_monitor(Func), Func} || Func <- Funcs], %% { {Pid, Ref}, Func}
    io:format("monitor_workers(): Workers: ~n~p~n", [Workers]),
    monitor_workers_loop(Workers).

monitor_workers_loop(Workers) ->
    receive
        {'DOWN', Ref, process, Pid, Why} ->
            io:format("monitor_workers_loop(): Worker ~w went down: ~w~n.", [{Pid, Ref}, Why]),
            NewWorkers = restart_worker({Pid, Ref}, Workers),
            monitor_workers_loop(NewWorkers);
        {request, stop} ->
            lists:foreach(fun({{Pid,_},_}) -> Pid ! stop end,  
                          Workers),  %% { {Pid, Ref}, Func}
            io:format("\tMonitor sent stop messages to all workers.~n"),
            io:format("\tMonitor terminating: normal.~n");
            
        {request, kill_worker, _From}  ->  %%For testing.
            kill_rand_worker(Workers),          
            monitor_workers_loop(Workers)           
    end.                                   

restart_worker(PidRef, Workers) ->  %%Monitor calls this function in response to a 'DOWN' message.
    {_, Func} = lists:keyfind(PidRef, 1, Workers),  %% { {Pid, Ref}, Func}
    NewPidRef = spawn_monitor(Func),          
    io:format("...restarting ~w => ~w) ~n", [PidRef, NewPidRef]),

    NewWorkers = lists:keydelete(PidRef, 1, Workers), %% { {Pid, Ref}, Func}
    [{NewPidRef, Func} | NewWorkers].
    
kill_rand_worker(Workers) ->   %%Monitor calls this function in response to a kill_worker request
    RandNum = rand:uniform(length(Workers) ),
    {{Pid, _}, _} = lists:nth(RandNum, Workers),  %% { {Pid, Ref} Func}
    io:format("kill_rand_worker(): about to kill ~w~n", [Pid]),
    exit(Pid, kill).
    
stop_monitor(Monitor) ->
    Monitor ! {request, stop}.

%======== TESTS ==========

worker(N) ->
    receive
        stop -> 
            io:format("\tWorker~w stopping: normal.~n", [N]) 
    after N*1000 ->
            io:format("Worker~w (~w) is still alive.~n", [N, self()] ),
            worker(N)
    end.

test() ->
    timer:sleep(500),  %%Allow output from startup of the erlang shell to print.

    Funcs = [fun() -> worker(N) end || N <- lists:seq(1, 4) ],
    Monitor = monitor_workers_init(Funcs),
    io:format("Monitor is: ~w~n", [Monitor]),

    timer:sleep(5200), %%Let monitored processes run for awhile.
    
    FiveTimes = 5,
    TimeBetweenKillings = 5200,
    kill_rand_worker(FiveTimes, TimeBetweenKillings, Monitor),

    stop_monitor(Monitor).

kill_rand_worker(0, _, _) ->
    ok;
kill_rand_worker(NumTimes, TimeBetweenKillings, Monitor) ->
    Monitor ! {request, kill_worker, self()},
    timer:sleep(TimeBetweenKillings),
    kill_rand_worker(NumTimes-1, TimeBetweenKillings, Monitor).
```
In the shell:
```
$ ./run
Erlang/OTP 19 [erts-8.2] [source] [64-bit] [smp:4:4] [async-threads:10] [hipe] [kernel-poll:false]
Eshell V8.2  (abort with ^G)

1> Monitor is: <0.59.0>
monitor_workers(): Workers: 
[{{<0.60.0>,#Ref<0.0.3.132>},#Fun<e5.1.131436102>},
 {{<0.61.0>,#Ref<0.0.3.133>},#Fun<e5.1.131436102>},
 {{<0.62.0>,#Ref<0.0.3.134>},#Fun<e5.1.131436102>},
 {{<0.63.0>,#Ref<0.0.3.135>},#Fun<e5.1.131436102>}]
Worker1 (<0.60.0>) is still alive.
Worker2 (<0.61.0>) is still alive.
Worker1 (<0.60.0>) is still alive.
Worker3 (<0.62.0>) is still alive.
Worker1 (<0.60.0>) is still alive.
Worker4 (<0.63.0>) is still alive.
Worker2 (<0.61.0>) is still alive.
Worker1 (<0.60.0>) is still alive.
Worker1 (<0.60.0>) is still alive.
kill_rand_worker(): about to kill <0.63.0>
monitor_loop(): Worker {<0.63.0>,#Ref<0.0.3.135>} went down: killed
....restarting {<0.63.0>,#Ref<0.0.3.135>} => {<0.64.0>,#Ref<0.0.3.150>}) 
Worker3 (<0.62.0>) is still alive.
Worker2 (<0.61.0>) is still alive.
Worker1 (<0.60.0>) is still alive.
Worker1 (<0.60.0>) is still alive.
Worker2 (<0.61.0>) is still alive.
Worker1 (<0.60.0>) is still alive.
Worker3 (<0.62.0>) is still alive.
Worker1 (<0.60.0>) is still alive.
Worker4 (<0.64.0>) is still alive.
Worker2 (<0.61.0>) is still alive.
Worker1 (<0.60.0>) is still alive.
kill_rand_worker(): about to kill <0.60.0>
monitor_loop(): Worker {<0.60.0>,#Ref<0.0.3.132>} went down: killed
....restarting {<0.60.0>,#Ref<0.0.3.132>} => {<0.65.0>,#Ref<0.0.3.165>}) 
Worker1 (<0.65.0>) is still alive.
Worker3 (<0.62.0>) is still alive.
Worker2 (<0.61.0>) is still alive.
Worker1 (<0.65.0>) is still alive.
Worker4 (<0.64.0>) is still alive.
Worker1 (<0.65.0>) is still alive.
Worker2 (<0.61.0>) is still alive.
Worker1 (<0.65.0>) is still alive.
Worker3 (<0.62.0>) is still alive.
Worker1 (<0.65.0>) is still alive.
kill_rand_worker(): about to kill <0.62.0>
monitor_loop(): Worker {<0.62.0>,#Ref<0.0.3.134>} went down: killed
....restarting {<0.62.0>,#Ref<0.0.3.134>} => {<0.66.0>,#Ref<0.0.3.179>}) 
Worker2 (<0.61.0>) is still alive.
Worker1 (<0.65.0>) is still alive.
Worker4 (<0.64.0>) is still alive.
Worker1 (<0.65.0>) is still alive.
Worker2 (<0.61.0>) is still alive.
Worker1 (<0.65.0>) is still alive.
Worker3 (<0.66.0>) is still alive.
Worker1 (<0.65.0>) is still alive.
Worker2 (<0.61.0>) is still alive.
Worker1 (<0.65.0>) is still alive.
kill_rand_worker(): about to kill <0.64.0>
monitor_loop(): Worker {<0.64.0>,#Ref<0.0.3.150>} went down: killed
....restarting {<0.64.0>,#Ref<0.0.3.150>} => {<0.67.0>,#Ref<0.0.3.193>}) 
Worker1 (<0.65.0>) is still alive.
Worker3 (<0.66.0>) is still alive.
Worker2 (<0.61.0>) is still alive.
Worker1 (<0.65.0>) is still alive.
Worker1 (<0.65.0>) is still alive.
Worker2 (<0.61.0>) is still alive.
Worker1 (<0.65.0>) is still alive.
Worker3 (<0.66.0>) is still alive.
Worker4 (<0.67.0>) is still alive.
Worker1 (<0.65.0>) is still alive.
kill_rand_worker(): about to kill <0.67.0>
monitor_loop(): Worker {<0.67.0>,#Ref<0.0.3.193>} went down: killed
....restarting {<0.67.0>,#Ref<0.0.3.193>} => {<0.68.0>,#Ref<0.0.3.207>}) 
Worker2 (<0.61.0>) is still alive.
Worker1 (<0.65.0>) is still alive.
Worker1 (<0.65.0>) is still alive.
Worker3 (<0.66.0>) is still alive.
Worker2 (<0.61.0>) is still alive.
Worker1 (<0.65.0>) is still alive.
Worker1 (<0.65.0>) is still alive.
Worker4 (<0.68.0>) is still alive.
Worker2 (<0.61.0>) is still alive.
Worker1 (<0.65.0>) is still alive.
Worker3 (<0.66.0>) is still alive.
        Monitor sent stop messages to all workers.
        Worker4 stopping: normal.
        Worker3 stopping: normal.
        Worker1 stopping: normal.
        Worker2 stopping: normal.
        Monitor terminating: normal.
```
There are a couple of things in the program output that demonstrate that eveything is working correctly:

1.  The worker numbers 1-4 appear in every secton of the output.

2.  In the restart output, you can actually examine the pid of the process being killed and look for the corresponding Worker number in the prior section of the output, then make a note of that Worker number.  Then examine the pid of the new process in the restart output and in the subsequent output check if the new pid corresponds to the same Worker number.

Checking to make sure that none of the processes that were started are still alive:
```
i().
Pid                   Initial Call                          Heap     Reds Msgs
Registered            Current Function                     Stack              
<0.0.0>               otp_ring0:start/2                      376      637    0
init                  init:loop/1                              2              
<0.1.0>               erts_code_purger:start/0               233        4    0
erts_code_purger      erts_code_purger:loop/0                  3              
<0.4.0>               erlang:apply/2                        2586   119174    0
erl_prim_loader       erl_prim_loader:loop/3                   5              
<0.30.0>              gen_event:init_it/6                    610      226    0
error_logger          gen_event:fetch_msg/5                    8              
<0.31.0>              erlang:apply/2                        1598      416    0
application_controlle gen_server:loop/6                        7              
<0.33.0>              application_master:init/4              233       64    0
                      application_master:main_loop/2           6              
<0.34.0>              application_master:start_it/4          233       59    0
                      application_master:loop_it/4             5              
<0.35.0>              supervisor:kernel/1                    610     1765    0
kernel_sup            gen_server:loop/6                        9              
<0.36.0>              erlang:apply/2                        4185   107483    0
code_server           code_server:loop/1                       3              
<0.38.0>              rpc:init/1                             233       21    0
rex                   gen_server:loop/6                        9              
<0.39.0>              global:init/1                          233       44    0
global_name_server    gen_server:loop/6                        9              
<0.40.0>              erlang:apply/2                         233       21    0
                      global:loop_the_locker/1                 5              
<0.41.0>              erlang:apply/2                         233        3    0
                      global:loop_the_registrar/0              2              
<0.42.0>              inet_db:init/1                         233      249    0
inet_db               gen_server:loop/6                        9              
<0.43.0>              global_group:init/1                    233       55    0
global_group          gen_server:loop/6                        9              
<0.44.0>              file_server:init/1                     233       78    0
file_server_2         gen_server:loop/6                        9              
<0.45.0>              supervisor_bridge:standard_error/      233       34    0
standard_error_sup    gen_server:loop/6                        9              
<0.46.0>              erlang:apply/2                         233       10    0
standard_error        standard_error:server_loop/1             2              
<0.47.0>              supervisor_bridge:user_sup/1           233       53    0
                      gen_server:loop/6                        9              
<0.48.0>              user_drv:server/2                     1598     4452    0
user_drv              user_drv:server_loop/6                   9              
<0.49.0>              group:server/3                         610    12679    0
user                  group:server_loop/3                      4              
<0.50.0>              group:server/3                         987    12510    0
                      group:server_loop/3                      4              
<0.51.0>              erlang:apply/2                        4185     9788    0
                      shell:shell_rep/4                       17              
<0.52.0>              kernel_config:init/1                   233      258    0
                      gen_server:loop/6                        9              
<0.53.0>              supervisor:kernel/1                    233       56    0
kernel_safe_sup       gen_server:loop/6                        9              
<0.57.0>              erlang:apply/2                        2586    18839    0
                      c:pinfo/1                               50              
Total                                                      23426   288978    0
                                                             222              
ok

2> 
```
No processes from e5 in there!

Below is a version that uses maps in Erlang 19.2.  It also employs `lists:foldl()` which is very similar to `lists:map()` in that they both apply a function to each element in a list.  With `lists:foldl()`, you manipulate the second argument in the body of the function, and you pass the altered second argument to the next function call.  After the function has been applied to the last element in the list, the manipulated second argument becomes the return value of `lists:foldl()`.  In my case, the manipulated second argument is a map, and every time the function is applied to an element of a list, a new entry is entered into the map.  After the function is applied to the last element in the list, the map is returned by `lists:foldl()`.

The difference between `foldl()` and `foldr()` is that `foldr()` applies a function to a list starting on the _right_ end of the list, i.e. the last element of the list, and steps through the list until it reaches the first element in the list.  Sometimes the ordering of computations or results is important, so you can choose the version of fold that provides the ordering you desire. 

```erlang
-module(e5).
%%-compile(export_all).
-export([monitor_workers_init/1, monitor_workers/1]).
-export([stop_monitor/1, test/0]).

monitor_workers_init(Funcs) ->
    spawn(?MODULE, monitor_workers, [Funcs]).

monitor_workers(Funcs) ->
    Workers = lists:foldl( 
                fun(Func, Map) ->  
                        Map#{spawn_monitor(Func) => Func}  %%{Pid, Ref} => Func
                end,
                #{}, Funcs),  %% An empty map is the initial value used for Map in the fun.
    io:format("moniter_workers(): Workers: ~n~p~n", [Workers]),
    monitor_workers_loop(Workers).

monitor_workers_loop(Workers) ->
    receive
        {'DOWN', Ref, process, Pid, Why} ->
            io:format("monitor_workers_loop(): Worker ~w went down: ~w~n.", [{Pid, Ref}, Why]),
            NewWorkers = restart_worker({Pid, Ref}, Workers),
            monitor_workers_loop(NewWorkers);
        {request, stop} ->
            lists:foreach(fun({Pid,_}) -> Pid ! stop end,  
                          maps:keys(Workers) ),  %% {Pid, Ref} => Func
            io:format("monitor_workers_loop():~n"),
            io:format("\tMonitor finished shutting down workers.~n"),
            io:format("\tMonitor terminating normally.~n");
        {request, current_workers, From} -> 
            From ! {reply, Workers, self()},
            monitor_workers_loop(Workers)
    end.

restart_worker(PidRef, Workers) ->
    #{PidRef := Func} = Workers,  %%PidRef and Workers are bound!
    NewPidRef = spawn_monitor(Func),          
    io:format("...restarting ~w => ~w) ~n", [PidRef, NewPidRef]),
    NewWorkers = maps:remove(PidRef, Workers),
    NewWorkers#{NewPidRef => Func}.
    
stop_monitor(Monitor) ->
    Monitor ! {request, stop}.

%======== TESTS ==========

worker(N) ->
    receive
        stop -> ok
    after N*1000 ->
            io:format("Worker~w (~w) is still alive.~n", [N, self()] ),
            worker(N)
    end.

test() ->
    timer:sleep(500),  %%Allow output from startup of the erlang shell to print.

    Funcs = [fun() -> worker(N) end || N <- lists:seq(1, 4) ],
    Monitor = monitor_workers_init(Funcs),
    io:format("Monitor is: ~w~n", [Monitor]),

    timer:sleep(5200), %%Let monitored processes run for awhile.

    FiveTimes = 5,
    TimeBetweenKillings = 5200,
    kill_rand_worker(FiveTimes, TimeBetweenKillings, Monitor),

    stop_monitor(Monitor).

kill_rand_worker(0, _, _) ->
    ok;
kill_rand_worker(NumTimes, TimeBetweenKillings, Monitor) ->
    Workers = get_workers(Monitor), 
    kill_rand_worker(Workers),
    timer:sleep(TimeBetweenKillings),
    kill_rand_worker(NumTimes-1, TimeBetweenKillings, Monitor).

get_workers(Monitor) ->
    Monitor ! {request, current_workers, self()},
    receive
        {reply, CurrentWorkers, Monitor} -> 
            CurrentWorkers
    end.

kill_rand_worker(Workers) ->
    {Pid, _} = get_random_key(Workers),
    io:format("kill_rand_worker(): about to kill ~w~n", [Pid]),
    exit(Pid, kill).

get_random_key(Map) ->
    Keys = maps:keys(Map),
    RandNum = rand:uniform(length(Keys) ),
    lists:nth(RandNum, Keys).

```
