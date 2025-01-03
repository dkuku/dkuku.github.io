---
title: "Show Country Emoji with Elixir"
meta_title: ""
description: "Recently jorik posted about converting country code to flag emoji. I decided to give it a try in elixir..."
date: 2022-01-08T09:57:16Z
image: "/images/image-placeholder.png"
categories: ["Development", "Tips"]
author: "Daniel Kukula"
tags: ["elixir", "emoji"]
draft: false
---

Recently [jorik](https://dev.to/jorik) [posted](https://dev.to/jorik/country-code-to-flag-emoji-a21) about converting country code to flag emoji. I decided to give it a try in elixir.

My first attempt:

```elixir
defmodule Flags do
  @flag_offset 127397
  def get_flag(country_code) do
    country_code
    |> String.upcase()
    |> String.split("", trim: true)  
    |> Enum.map(fn <<string::utf8>> -> <<(string + @flag_offset)::utf8>> end)
    |> Enum.join("")
  end
end

~w(us gb de pl) |> Enum.map(&Flags.get_flag()/1) |> Enum.join()
"🇺🇸🇬🇧🇩🇪🇵🇱"
```

But can it be simplified? Sure it can:

```elixir
defmodule Flags do
  @flag_offset 127397
  def get_flag(country_code) when byte_size(country_code) == 2 do
    <<s1::utf8, s2::utf8>> = String.upcase(country_code)  
    <<(s1 + @flag_offset)::utf8, (s2 + @flag_offset)::utf8>>
  end
  def get_flag(country_code), do: country_code
end

~w(us gb pl US) |> Enum.map(&Flags.get_flag()/1) |> Enum.join()
"🇺🇸🇬🇧🇵🇱🇺🇸"
```
