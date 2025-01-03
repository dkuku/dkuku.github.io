---
title: "Enhancements to dbg in Elixir 1.18"
meta_title: ""
description: "Elixir 1.18 added some interesting features, but one that went under the radar was extended support for dbg..."
date: 2024-12-22T08:28:30Z
image: "/images/image-placeholder.png"
categories: ["Development", "Tutorial"]
author: "Daniel Kukula"
tags: ["elixir", "debugging", "programming"]
draft: false
---

Elixir 1.18 added some interesting features, but one that went under the radar was extended support for `dbg`. In 1.17 when you had this code:

```elixir
a = 1
b = 2

if a + b == 3 do
  :equal
else
  :non_equal
end
|> dbg
```

It evaluated to:

```elixir
if a + b == 3 do
  :equal
else
  :non_equal
end #=> :equal
```

In 1.18 this was aligned with what `case` and `cond` does:

```elixir
Case argument:
1 + 2 #=> 3

Case expression (clause #1 matched):
case 1 + 2 do
  3 -> :equal
  _ -> :non_equal
end #=> :equal
```

`if` result in 1.18:
```elixir
If condition:
a + b == 3 #=> true

If expression:
if a + b == 3 do
  :equal
else
  :non_equal
end #=> :equal
```

If this is not enough you can wrap your code in brackets and pass it to dbg:

```elixir
(
  input = %{a: 1, b: 2}
  c = Map.get(input, :c, 20)

  if input[:a] + c == 3 do
    :equal
  else
    :non_equal
  end
)
|> dbg
```

Results in displaying results for every expression:

```elixir
Code block:
(
  input = %{a: 1, b: 2} #=> %{a: 1, b: 2}
  c = Map.get(input, :c, 20) #=> 20
  if input[:a] + c == 3 do
  :equal
else
  :non_equal
end #=> :non_equal
)
```

Last missing piece is the `with` macro which is sometimes a nightmare to debug, in Elixir 1.18 we have support for that:

```elixir
with input = %{a: 1, b: 2},
     {:ok, c} <- Map.fetch(input, :c),
     3 <- input[:a] + c do
  :equal
else
  _ -> :non_equal
end
|> dbg
```

The result here is:

```elixir
With clauses:
%{a: 1, b: 2} #=> %{a: 1, b: 2}
Map.fetch(input, :c) #=> :error

With expression:
with input = %{a: 1, b: 2},
     {:ok, c} <- Map.fetch(input, :c),
     3 <- input[:a] + c do
  :equal
else
  _ -> :non_equal
end #=> :non_equal
```

This should make your debugging journey easier. 
If you are stuck in older Elixir versions you can install a package that backports this functionality [dbg_mate](https://hex.pm/packages/dbg_mate).
Thanks for your attention.
