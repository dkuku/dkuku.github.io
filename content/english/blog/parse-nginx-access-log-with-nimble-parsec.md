---
title: "Parse Nginx Access Log with Nimble Parsec"
meta_title: ""
description: "Lately I played with some string parsing and found nimble_parsec a library which does the job perfect..."
date: 2020-12-29T12:50:02Z
image: "/images/image-placeholder.png"
categories: ["Development", "Tutorials"]
author: "Daniel Kukula"
tags: ["elixir", "nginx", "parsing"]
draft: false
---

Lately I played with some string parsing and found `nimble_parsec` a library which does the job perfect. I will show you how to convert nginx logs to something readable in elixir. A typical log line looks like:

```
127.0.0.1 - - [25/Dec/2020:08:15:53 +0000] 
"GET /img/key.svg HTTP/1.1" 200 8305 
"http://localhost/styles/css/app.css" 
"Mozilla/5.0 Firefox/84.0"
```

These fields are: 
```
remote_addr - remote_user [time_local] 
"request" status bytes_sent 
"referer" 
"user_agent"
```

I found the description on [stack overflow](https://stackoverflow.com/questions/26780466/nginx-understanding-access-log-column)

To start matching on our string we need to define our parser in a new file:
```elixir
defmodule NginxParsex do
  import NimbleParsec

  defparsec( :ngnix_parser, integer(1))
end
```

This just matches a string starting with an integer:
```elixir
iex(1)> log = "127.0.0.1 - - "
iex(2)> NginxParsex.ngnix_parser(log)
{:ok, [1], "27.0.0.1 - - ", %{}, {1, 0}, 1}
```

What happens here is our parser matches the first integer and returns a tuple with `:ok`, a list of items that matched [1], and the rest of the string. The other items in the tuple are additional data used by the parser.

We can extract our full IP address creating a parser like this: 
```elixir
  ip = 
    integer(min: 1, max: 3)
    |> ignore(string("."))
    |> integer(min: 1, max: 3)
    |> ignore(string("."))
    |> integer(min: 1, max: 3)
    |> ignore(string("."))
    |> integer(min: 1, max: 3)

  defparsec( :ngnix_parser, ip)
```

And after running it we got:
```elixir
iex(1)> NginxParsex.ngnix_parser(log)
{:ok, [127, 0, 0, 1], " - - ", %{}, {1, 0}, 9}
```

As you can see we are extracting all the numbers and omitting the dots. As we don't need all the integers, a string is enough so we may also extract it as a string that consists of numbers and dots:
```elixir
ip = ascii_string([?., ?0..?9], min: 7, max: 15) 
```

Now we get back:
```elixir
iex(1)> NginxParsex.ngnix_parser(log) 
{:ok, ["127.0.0.1"], " - - ", %{}, {1, 0}, 9}
```

The next part is not really useful for us `" - - "` we may either omit it using the ignore macro:
```elixir
  ip = 
    ascii_string([?., ?0..?9], min: 7, max: 15)
    |> ignore(string(" - - "))
```

And our first test string is parsed:
```elixir
iex(1)> NginxParsex.ngnix_parser(log)
{:ok, ["127.0.0.1"], "", %{}, {1, 0}, 14}
```

The problem is the second dash may be a `remote_user` which we will skip and match on the opening `[`:
```elixir
  ip = 
    ascii_string([?., ?0..?9], min: 7, max: 15)
    |> ignore(eventually(ascii_char([?[])))
```

As our current log was missing this part I'll add it:
```elixir
log = ~s(127.0.0.1 - - [25/Dec/2020:08:15:53 +0000] )
iex(1)> NginxParsex.ngnix_parser(log)
{:ok, ["127.0.0.1"], "25/Dec/2020:08:15:53 +0000]", %{}, {1, 0}, 15}
```

Next part we need to match is the date string - this is very well explained in the [documentation](https://hexdocs.pm/nimble_parsec/NimbleParsec.html#module-examples) we just need to make some adjustments. Also we will extract our date and time parsers. Now our module looks like this:
```elixir
defmodule NginxParsex do
  import NimbleParsec

  ip = 
    ascii_string([?., ?0..?9], min: 7, max: 15)

  date =
    integer(2)
    |> ignore(string("/"))
    |> ascii_string([?a..?z, ?A..?Z], 3)
    |> ignore(string("/"))
    |> integer(4)

  time =
    integer(2)
    |> ignore(string(":"))
    |> integer(2)
    |> ignore(string(":"))
    |> integer(2)
    |> ignore(string(" "))
    |> ignore(ascii_char([?-, ?+]))
    |> ignore(integer(4))

  defparsec( :ngnix_parser,
    ip
    |> ignore(eventually(ascii_char([?[])))
    |> concat(date)
    |> ignore(string(":"))
    |> concat(time)
    |> ignore(string("] "))
  )
end
```

When running the code we got:
```elixir
iex(1)> NginxParsex.ngnix_parser(log)                        
{:ok, ["127.0.0.1", 25, "Dec", 2020, 8, 15, 53], "", %{}, {1, 0}, 43}
```

Cool our date and time is parsed, we can add more stuff from the line:
```elixir
log = ~s(127.0.0.1 - - [25/Dec/2020:08:15:53 +0000] "GET /img/key.svg HTTP/1.1" 200 8305 "http://localhost/styles/css/app.css" "Mozilla/5.0 Firefox/84.0")
```

Now we need to match the string inside quotes:
```elixir
  string_in_quotes =
    ignore(ascii_char([?"]))
    |> ascii_string([not: ?"], min: 1)
    |> ignore(ascii_char([?"]))

  defparsec( :ngnix_parser,
    ip
    |> ignore(eventually(ascii_char([?[])))
    ...
    |> concat(string_in_quotes)
  )
```

Our result:
```elixir
NginxParsex.ngnix_parser(log)                                                                            
{:ok, ["127.0.0.1", 25, "Dec", 2020, 8, 15, 53, "GET /img/key.svg HTTP/1.1"],
 " 200 8305 \"http://localhost/styles/css/app.css\" \"Mozilla/5.0 Firefox/84.0\"",
 %{}, {1, 0}, 70}
```

What's left are some spaces, numbers and two quoted strings - we can reuse the parts we already have and our full parser looks now:
```elixir
  defparsec( :ngnix_parser,
    ip
    |> ignore(eventually(ascii_char([?[])))
    |> concat(date)
    |> ignore(string(":"))
    |> concat(time)
    |> ignore(string("] "))
    |> concat(string_in_quotes)
    |> ignore(string(" "))
    |> integer(min: 1)
    |> ignore(string(" "))
    |> integer(min: 1)
    |> ignore(string(" "))
    |> concat(string_in_quotes)
    |> ignore(string(" "))
    |> concat(string_in_quotes)
```

The result is now:
```elixir
{:ok,
 ["127.0.0.1", 25, "Dec", 2020, 8, 15, 53, "GET /img/key.svg HTTP/1.1", 200,
  8305, "http://localhost/styles/css/app.css", "Mozilla/5.0 Firefox/84.0"], "",
 %{}, {1, 0}, 144}
```

To retrieve the data we can pattern match on the result:
```elixir
  {:ok, 
[ ip, day, month, year, hour, minute, seconds, request, code, size, referrer, user_agent ],
 _, _, _, _} = NginxParsex.ngnix_parser(log)
{:ok,
 ["127.0.0.1", 25, "Dec", 2020, 8, 15, 53, "GET /img/key.svg HTTP/1.1", 200,
  8305, "http://localhost/styles/css/app.css", "Mozilla/5.0 Firefox/84.0"], "",
 %{}, {1, 0}, 144}
iex(111)> user_agent
"Mozilla/5.0 Firefox/84.0"
```

Now when we got all variables we can process them and wrap it in a map:
```elixir
  @month_map %{
    "Jan" => 1,
    "Feb" => 2,
    "Mar" => 3,
    "Apr" => 4,
    "May" => 5,
    "Jun" => 6,
    "Jul" => 7,
    "Aug" => 8,
    "Oct" => 9,
    "Sep" => 10,
    "Nov" => 11,
    "Dec" => 12
  }
  %{
    ip: ip,
    date: Date.new!(year, @month_map[month], day),
    time: Time.new!(hour, minute, seconds),
    request: request,
    code: code,
    size: size,
    referrer: URI.decode(referrer),
    user_agent: user_agent
 }
%{
  code: 200,
  date: ~D[2020-12-25],
  ip: "127.0.0.1",
  referrer: "http://localhost/styles/css/app.css",
  request: "GET /img/key.svg HTTP/1.1",
  size: 8305,
  time: ~T[08:15:53],
  user_agent: "Mozilla/5.0 Firefox/84.0"
}
```

Thanks for reading - I tested it on a 2MB file that I have on my local machine and it can parse it all to the end.
A full file we wrote today can be found in this [gist](https://gist.github.com/dkuku/87fc8da8bc62c9161b6f9b98fb7a91e6).
