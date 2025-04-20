## Typespecs - best way to spec keyword lists?

https://elixirforum.com/t/typespecs-best-way-to-spec-keyword-lists/2991/1
or `Examples of composing a keyword list type`
https://github.com/elixir-lang/elixir/pull/12482

> (Dec/2016)

The [Elixir Typespec docs][1] show the following syntax for keyword lists in typespecs:

```
# ...
| [key: type]                   # keyword lists
# ...
```

It's nice to have a syntax that is like the literal keyword list syntax,
but there are a number of open questions here:


  Given that lists are ordered, and `[a: 1, b: 2] = kw_list` only matches if
  kw_list is in that order... is a typespec like `[a: integer, b: integer]`
  only satisfied by a keyword list in that order?
  Are all keys required? Or are all keys optional?

Elsewhere I've seen keyword lists type spec'd doing something like:

```elixir
@type option :: {:name, String.t} | {:max, pos_integer} | {:min, pos_integer}
@type options :: [option]
```

This is very clear: I can tell that order does not manner, and
no options are required.

Can someone from Elixir Core weigh in?
I'm getting dialyzer running on an existing application and
am trying to understand the right way to spec keyword lists
(and also to be sure I'm satisfying the typespecs in Elixir itself).

[1]: http://elixir-lang.org/docs/v1.3/elixir/typespecs.html



## Answers:

> michalmuskala:

The longer form of separated option and options type
has a very important advantage - it's composable.
It allows to write code like this:

```elixir
  @type option :: {:my_option, String.t} | GenServer.option

  @spec start_link([option]) :: GenServer.on_start
  def start_link(opts) do
    {my_opts, gen_server_opts} = Keyword.split(opts, [:my_option])
    GenServer.start_link(__MODULE__, my_opts, gen_server_opts)
  end
```

That kind of extension is not possible when you have a full list type.


> jwarlander:

Some quick tests:

```elixir
defmodule TypeSpecDemo do
  @moduledoc """
  Documentation for TypeSpecDemo.
  """

  @spec hello([bar: String.t, baaz: String.t]) :: {:world, list}
  def hello(opts \\ []) do
    {:world, opts}
  end

  # correct usage
  def default_to_empty_list, do: hello()
  def call_with_empty_list, do: hello([])
  def first_key_only, do: hello(bar: "bar")
  def second_key_only, do: hello([baaz: "baaz"])
  def both_keys_in_order, do: hello([bar: "bar", baaz: "baaz"])
  def both_keys_reversed, do: hello([baaz: "baaz", bar: "bar"])

  # incorrect usage
  def bad_arg, do: hello("world")
  def unknown_key, do: hello(foo: "foo")
  def wrong_value, do: hello(baaz: 15)
end
```

Setting up according to instructions for Dialyxir, and then
running `mix dialyzer`, gives me the following output:

```
Compiling 1 file (.ex)
Checking PLT...
[:compiler, :elixir, :kernel, :logger, :stdlib]
PLT is up to date!
Starting Dialyzer
dialyzer --no_check_plt --plt /home/johwar/src/type_spec_demo/_build/dev/dialyxir_erlang-18.2.1_elixir-1.4.0-rc.1_deps-dev.plt /home/johwar/src/type_spec_demo/_build/dev/lib/type_spec_demo/ebin
  Proceeding with analysis...
type_spec_demo.ex:21: Function bad_arg/0 has no local return
type_spec_demo.ex:21: The call 'Elixir.TypeSpecDemo':hello(<<_:40>>) breaks the contract ([{'bar','Elixir.String':t()} | {'baaz','Elixir.String':t()}]) -> {'world',[any()]}
type_spec_demo.ex:22: Function unknown_key/0 has no local return
type_spec_demo.ex:22: The call 'Elixir.TypeSpecDemo':hello([{'foo',<<_:24>>},...]) breaks the contract ([{'bar','Elixir.String':t()} | {'baaz','Elixir.String':t()}]) -> {'world',[any()]}
type_spec_demo.ex:23: Function wrong_value/0 has no local return
type_spec_demo.ex:23: The call 'Elixir.TypeSpecDemo':hello([{'baaz',15},...]) breaks the contract ([{'bar','Elixir.String':t()} | {'baaz','Elixir.String':t()}]) -> {'world',[any()]}
 done in 0m1.04s
done (warnings were emitted)
```

It seems that the shorter, documented syntax of `[key1: type1, key2: type2]`
actually means the same as `[{:key1, type1} | {:key2, type2}]`.
Order does not matter and an empty list is OK, but unknown keys are rejected.

Maybe there are situations where
it's useful to have a type for the individual allowed keyword entries I guess,
like option in our example, but until I run across that I'd probably use the
shorter syntax.
