---
title: "Reverse Engineer Unknown Data Format Using Elixir"
meta_title: ""
description: "I recently worked on porting the csv file converter for AdventureWorks database from ruby to elixir..."
date: 2023-01-10T18:37:35Z
image: "https://dev-to-uploads.s3.amazonaws.com/uploads/articles/h4p3h635gx967arkv8od.png"
categories: ["Development", "Tutorial"]
author: "Daniel Kukula"
tags: ["elixir", "reverse-engineering", "postgresql"]
draft: false
---

I recently worked on porting the csv file converter for AdventureWorks database from [ruby to elixir](https://github.com/dkuku/AdventureWorks-for-Postgres). This ended successful and I have all the data in my local postgresql to play with. By browsing the tables I found `spatial_location` column in `person.address` table and I wanted to decode it.

```
|address_line1       |city   |spatial_location                            |
|--------------------|-------|-------------------------------------------|
|1970 Napa Ct.       |Bothell|E6100000010CAE8BFC28BCE4474067A89189898A5EC0|
|9833 Mt. Dias Blv.  |Bothell|E6100000010CD6FA851AE6D74740BC262A0A03905EC0|
|7484 Roundtree Drive|Bothell|E6100000010C18E304C4ADE14740DA930C7893915EC0|
|9539 Glenside Dr    |Bothell|E6100000010C813A0D5F9FDE474011A5C28A7C955EC0|
|1226 Shoe St.       |Bothell|E6100000010C61C64D8ABBD94740C460EA3FD8855EC0|
|1399 Firestone Drive|Bothell|E6100000010CE0B4E50458DA47402F12A5F80C975EC0|
|5672 Hale Dr.       |Bothell|E6100000010C18E304C4ADE1474011A5C28A7C955EC0|
|6387 Scenic Avenue  |Bothell|E6100000010C0029A5D93BDF4740E248962FD5975EC0|
|8713 Yosemite Ct.   |Bothell|E6100000010C6A80AD742DDC4740851574F7198C5EC0|
```

As you can see the data looks like binary data.
I found even some [documentation from microsoft](https://learn.microsoft.com/en-us/openspecs/sql_server_protocols/ms-ssclrt/77460aa9-8c2f-4449-a65e-1d649ebd77fa) that may be useful here but in the end I could not use it with luck. 
So what I end up doing to decode it.

First what I know and may be useful:
- I got the city and address
- I know that the data is binary encoded as string because there is nothing higher than F.
- The string is 44 letters long - every byte is encoded using 2 letters so I have 22 bytes of data

Let's jump to iex:
```elixir
iex(1)> encoded = "E6100000010C6A80AD742DDC4740851574F7198C5EC0"
"E6100000010C6A80AD742DDC4740851574F7198C5EC0"
iex(2)> String.length(encoded)
44
iex(3)> decoded = Base.decode16!(encoded)
<<230, 16, 0, 0, 1, 12, 106, 128, 173, 116, 45, 220, 71, 64, 133, 21, 116, 247,
  25, 140, 94, 192>>
```

Next thing that I did was to find what exactly can be hidden in my data:
The beginning and end look the same between the first lines I found - it may be some kind of wrapper and I may need to skip it.
I also searched for the actual coordinates of Bothell to see if I can see some similarities:

![Bothell coordinates](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/h4p3h635gx967arkv8od.png)

It looks like the data encoded is around `47.5` and `-122.5`
so let's move back to iex.

```elixir
iex(3)> decoded = Base.decode16!(encoded)
<<230, 16, 0, 0, 1, 12, 106, 128, 173, 116, 45, 220, 71, 64, 133, 21, 116, 247,
  25, 140, 94, 192>>
iex(4)> <<47.0::float>>
<<64, 71, 128, 0, 0, 0, 0, 0>>
iex(5)> <<47.99::float>>
<<64, 71, 254, 184, 81, 235, 133, 31>>
```

I tried how float 47 looks like in binary form. It starts with
`64, 71` and has 8 bytes. Looking at the encoded data I have 64 and even 71 - just in reverse order, is it a coincidence??
Let's try with -122

```elixir
<<230, 16, 0, 0, 1, 12, 106, 128, 173, 116, 45, 220, 71, 64, 133, 21, 116, 247,
  25, 140, 94, 192>>
iex(7)> <<-122.0::float>>                
<<192, 94, 128, 0, 0, 0, 0, 0>>
iex(8)> <<-122.9::float>>
<<192, 94, 185, 153, 153, 153, 153, 154>>
```

`192, 94` - again I can see similar pattern but in reverse order at the end.
It turns out that I can use the [little endian modifier](https://en.wikipedia.org/wiki/Endianness) to create similar looking data:

```elixir
iex(9)> <<-122.9::little-float>>
<<154, 153, 153, 153, 153, 185, 94, 192>>
```

Hmm - definitely looks similar. Especially that it's 8 numbers from the end and If I look at the encoded data then on the 9th position I have 64 which is the possible beginning of 47.0 encoded as binary. When removing this data I end up with 6 bits at the beginning. Back to iex:

```elixir
iex(10)> <<_::binary-size(6), x::little-float, y::little-float>> = decoded
<<230, 16, 0, 0, 1, 12, 106, 128, 173, 116, 45, 220, 71, 64, 133, 21, 116, 247,
  25, 140, 94, 192>>
iex(11)> x
47.7201372000862
iex(12)> y
-122.189084876407
```

Was it really so easy?? 
Close but I can't see the street `8713 Yosemite Ct.`

![First location on map](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/pw23baea6e2y98kbr90c.png)

Let's try with another one:
`6387 Scenic Avenue, Bothell`
```elixir
iex(15)> <<_::binary-size(6), x::little-float, y::little-float>> = 
iex(..)> Base.decode16!("E6100000010C0029A5D93BDF4740E248962FD5975EC0")  
<<230, 16, 0, 0, 1, 12, 0, 41, 165, 217, 59, 223, 71, 64, 226, 72, 150, 47, 213,
  151, 94, 192>>
iex(16)> x
47.7440139824339
iex(17)> y
-122.372386833918
```

![Second location on map](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ws3dvq3ohwqiep3topti.png)

Again it's not a complete hit. Let's see if we can be more specific?
From the ms docs I found that the encoded data starts with SRID
```
SRID (4 bytes): (32 bit integer) The SRID for the geography. GEOGRAPHY structures MUST use
SRID values in the range of 4120 through 4999, inclusive, with the exception of null geographies.
A value of -1 indicates a null geography. When a null geography is indicated, all other fields are
omitted. Default SRID for GEOGRAPHY instances is 4326. Default SRID for GEOMETRY instances is
zero (0). For GEOMETRY instance, SRID can be any value: SRID is not constrained.
```

Let's again try to decode it, this time it's an integer.

```elixir
iex(35)> <<srid::little-integer-size(32), _::binary>> = Base.decode16!("E6100000010C0029A5D93BDF4740E248962FD5975EC0")
<<230, 16, 0, 0, 1, 12, 0, 41, 165, 217, 59, 223, 71, 64, 226, 72, 150, 47, 213,
  151, 94, 192>>
iex(36)> srid
4326
``` 

This is in the 4120..4999 range and after short google search it turns out that it's a [standard encoding](https://epsg.io/4326) that is used by openstreetmap.
Let's check how far are we off at least?

![No result](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/k89rn1u36b6morzxalx8.png)

Ohh - maybe that's the problem? Let's try another one:

![No result](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/vlkaqq2bkxyd8l02o5nn.png)

:facepalm::facepalm::facepalm::facepalm::facepalm::facepalm::facepalm:

It turns out that the data is synthetic so the exact location won't be shown. But it does not mean that it was not worth checking out how to do this. I definitely learned something along the way and I hope so are you when reading it.
