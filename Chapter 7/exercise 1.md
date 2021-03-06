`reverse(Bin)`:

Constructing a binary is much like constructing a list, but instead of using cons, `|`, to add an element to a binary you use a comma: 

    << ItemToAdd, SomeBinary/binary >>
    
And when pattern matching binaries, the equivalent of:

     [H|T]
    
is: 
 
     <<H, T/binary>>

Here's an example in the shell:
```erlang
73> f().
ok

74> <<X, Rest/binary>> = <<6, 3, 9, 2>>.     
<<6,3,9,2>>

75> X.
6

76> Rest.
<<3,9,2>>
```

Or, you could do this:
```erlang
77> f().
ok

78> <<X:1/binary, Rest/binary>> = <<6, 3, 9, 2>>.     
<<6,3,9,2>>

79> X.
<<6>>

80> Rest.
<<3,9,2>>
```
Note that Size is specified in bits, but the *total size* of a segment is actually `Size * unit`, and for the binary Type the default for `unit` is 8, so the total size of X is `1*8 = 8 bits` (the default value for `unit` for the other Types is 1).  Another way to think about it is, for the binary type the Size is the number of *bytes* for the segment, and for the other types the Size is the number of *bits*.

Now, for the exercise:

```erlang
-module(bin).
-compile(export_all).

reverse_bytes(Bin) ->
    reverse_bytes(Bin, <<>>).
reverse_bytes(<<X, Rest/binary>>, Acc) ->    %When matching, if a binary Type is the last segment
    reverse_bytes(Rest, <<X, Acc/binary>>);  %its Size can be omitted, and its default Size will
reverse_bytes(<<>>, Acc) ->                  %be the rest of the binary that you are matching against.
    Acc.
```

Compare that to reversing a list:

```erlang
reverse_list(List) ->
    reverse_list(List, []).
reverse_list([X|Rest], Acc) ->
    reverse_list(Rest, [X|Acc]);
reverse_list([], Acc) ->
    Acc.
```

The default Type for a segment is `integer`, and the default Size for `integer` is 8 bits, so the example above is equivalent to:

```erlang
reverse_bytes(Bin) ->
    reverse_bytes(Bin, <<>>).
reverse_bytes(<<X:8/integer, Rest/binary>>, Acc) ->
    reverse_bytes(Rest, <<X:8/integer, Acc/binary>>);
reverse_bytes(<<>>, Acc) ->
    Acc.                         
```

In the shell:
```erlang
10> c(bin).
{ok,bin}

11> bin:reverse_bytes(<<1, 2, 3, 4>>).
<<4,3,2,1>>

12> bin:reverse_bytes(<<>>).
<<>>

13> bin:reverse_bytes(<<97, 98, 99>>).
<<"cba">>
```






