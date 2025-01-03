---
title: "Optimizing DateTime Serialization in Elixir"
meta_title: ""
description: "A deep dive into optimizing Elixir's Calendar module, improving datetime serialization performance through iodata and improper lists"
date: 2025-01-02T23:16:36+01:00
image: "/images/image-placeholder.png"
categories: ["Development", "Performance"]
author: "Daniel Kukula"
tags: ["elixir", "performance", "optimization", "calendar"]
draft: false
---

# The Journey of Optimization
## A deep dive into optimizing Elixir's Calendar module, improving datetime serialization performance through iodata and improper lists

Recently, I watched some Elixir vs Go comparison videos on YouTube. After the first comparison, José Valim made a PR to make the comparison more accurate. One key difference was that the Elixir version used Ecto and serialized datetime multiple times, while the Go version used raw SQL and single datetime serialization.

While these micro-benchmarks might not reflect real-world scenarios (where database operations typically account for 90% of response time), they sparked my curiosity about how Elixir handles datetime serialization internally.

## Initial Investigation: The String Concatenation Problem

Looking into Elixir's source code, specifically the `calendar/iso.ex` module, I noticed frequent use of the `<>` operator for string concatenation. Here's a simple example of how string concatenation works:

```elixir
iex(1)> time = "10:10:10"
"10:10:10"
iex(2)> date = "2025-01-02"
"2025-01-02"
iex(3)> datetime = date <> " " <> time
"2025-01-02 10:10:10"
```

The problem? Each concatenation creates a new string, which is inefficient when we don't actually need intermediate strings. Erlang VM provides better tools for this: IOLists.

## Enter IOLists: A Better Approach

Instead of concatenating strings, we can use IOLists - a powerful BEAM feature that allows us to work with strings, charlists, and bytes without creating intermediate strings:

```elixir
iex(4)> IO.puts([date, " ", time])
2025-01-02 10:10:10
```

You can even use IOLists in string interpolation:

```elixir
iex(5)> "the time is #{[date, 32, time]}"
"the time is 2025-01-02 10:10:10"
```

The number 32 here represents a space character in ASCII - IOLists can contain integers (0-255) representing bytes.

## The Original Implementation

Here's a simplified version of the original Calendar module:

```elixir
defmodule LegacyCalendar do
  def datetime_to_string(dt) do
    %{
      year: year,
      month: month,
      day: day,
      hour: hour,
      minute: minute,
      second: second,
      microsecond: microsecond
    } = dt

    date_to_string(year, month, day) <>
      " " <>
      time_to_string(hour, minute, second, microsecond) <>
      "Z"
  end

  defp time_to_string(hour, minute, second, {microsecond, precision}) do
    time_to_string_format(hour, minute, second) <>
      "." <> (microsecond |> zero_pad(6) |> binary_part(0, precision))
  end

  defp time_to_string_format(hour, minute, second) do
    zero_pad(hour, 2) <> ":" <> zero_pad(minute, 2) <> ":" <> zero_pad(second, 2)
  end

  defp date_to_string(year, month, day) do
    zero_pad(year, 4) <> "-" <> zero_pad(month, 2) <> "-" <> zero_pad(day, 2)
  end

  defp zero_pad(val, count) when val >= 0 do
    num = Integer.to_string(val)
    :binary.copy("0", max(count - byte_size(num), 0)) <> num
  end

  defp zero_pad(val, count) do
    "-" <> zero_pad(-val, count)
  end
end
```

Running this 1000 times takes about 3ms:
```bash
mix run -e ":timer.tc(fn -> for _ <- 1..1000 do LegacyCalendar.datetime_to_string(~U[2025-01-02T21:22:23.012345Z]) end end) |> elem(0) |> IO.puts"
# Output: 3146ms
```

## Step 1: Converting to IOLists

The first optimization replaced string concatenations with IOLists:

```elixir
def datetime_to_string(dt) do
  [
    date_to_string(year, month, day),
    ?\s,
    time_to_string(hour, minute, second, microsecond),
    ?Z
  ]
  |> IO.iodata_to_binary()
end
```

This change reduced execution time to 2700ms.

## Step 2: Optimizing Microseconds

The next optimization targeted microsecond formatting:

```elixir
def microseconds_to_iodata(_microsecond, 0), do: []
def microseconds_to_iodata(microsecond, 6), do: zero_pad(microsecond, 6)

def microseconds_to_iodata(microsecond, precision) do
  num = div(microsecond, div_factor(precision))
  zero_pad(num, precision)
end
```

This brought the time down to 2500ms.

## Step 3: Optimizing Zero Padding

We optimized the zero padding function by precomputing binary parts:

```elixir
defp zero_pad(val, count) when val >= 0 and count <= 6 do
  num = Integer.to_string(val)
  case max(count - byte_size(num), 0) do
    0 -> num
    1 -> ["0", num]
    2 -> ["00", num]
    3 -> ["000", num]
    4 -> ["0000", num]
    5 -> ["00000", num]
  end
end
```

This reduced execution time to around 1900ms.

## Step 4: Improper Lists

Following a suggestion from @sabiwara, we implemented improper lists - a technique where the last element isn't an empty list but a string:

```elixir
[
  date_to_string(year, month, day),
  ?\s,
  time_to_string(hour, minute, second, microsecond)
  | "Z"
]
```

This optimization brought us down to 1800ms.

## Final Optimization: Special Cases

The last optimization added special cases for two-digit numbers, which are used frequently in time formatting:

```elixir
defp zero_pad(ones, 2) when ones >= 0 and ones < 10 do
  [?0, ones + ?0]
end

defp zero_pad(val, 2) when val >= 10 and val < 100 do
  tens = div(val, 10)
  ones = rem(val, 10)
  [tens + ?0, ones + ?0]
end
```

This brought our final execution time down to around 1714ms.

# Memory Benefits

Beyond performance improvements, returning IOLists directly (without converting to binary) can save memory when working with libraries that accept IOData (like Phoenix or JSON encoders). The string doesn't need to be created and later garbage collected.

# Conclusion

Through this optimization journey, we improved datetime serialization performance by:
1. Replacing string concatenation with IOLists
2. Optimizing microsecond handling
3. Using improper lists
4. Adding special cases for common patterns

The final result is not just faster execution (from 3146μs to 1714μs) but also more memory-efficient when used with IOData-aware libraries.

This exercise shows that even in a well-optimized language like Elixir, understanding the underlying BEAM features and applying them appropriately can lead to significant improvements.

## Benchee Output

Here are the Benchee outputs for each step of the optimization process:

### Initial Implementation

```
Name                     ips        average  deviation         median         99th %
LegacyCalendar        1.19 M      842.19 ns  ±2153.46%         551 ns        1362 ns

Extended statistics:

Name                   minimum        maximum    sample size                     mode
LegacyCalendar          470 ns    10294513 ns         1.81 M                   541 ns

Memory usage statistics:

Name              Memory usage
LegacyCalendar           896 B

**All measurements for memory usage were the same**

Profiling LegacyCalendar with eprof...

Profile results of #PID<0.488555.0>
#                                           CALLS     % TIME µS/CALL
Total                                          27 100.0   15    0.56
LegacyCalendar.time_to_string_format/3          1  0.00    0    0.00
LegacyCalendar.datetime_to_string/1             1  0.00    0    0.00
LegacyCalendar.date_to_string/3                 1  0.00    0    0.00
anonymous fn/0 in CalendarStringBench.run/0     1  0.00    0    0.00
:binary.copy/2                                  7  6.67    1    0.14
LegacyCalendar.time_to_string/4                 1 13.33    2    2.00
:erlang.integer_to_binary/1                     7 13.33    2    0.29
:erlang.apply/2                                 1 20.00    3    3.00
LegacyCalendar.zero_pad/2                       7 46.67    7    1.00

Profile done over 9 matching functions

```

### Optimized Implementation

```
Name                     ips        average  deviation         median         99th %
LegacyCalendar        3.75 M      266.66 ns  ±4073.37%         231 ns         361 ns

Extended statistics:

Name                   minimum        maximum    sample size                     mode
LegacyCalendar          200 ns    15489642 ns         4.68 M                   230 ns

Memory usage statistics:

Name              Memory usage
LegacyCalendar           440 B

**All measurements for memory usage were the same**

Profiling LegacyCalendar with eprof...

Profile results of #PID<0.561770.0>
#                                           CALLS     % TIME µS/CALL
Total                                          22 100.0    6    0.27
LegacyCalendar.time_to_string_format/3          1  0.00    0    0.00
LegacyCalendar.time_to_string/4                 1  0.00    0    0.00
LegacyCalendar.microseconds_to_iodata/2         1  0.00    0    0.00
LegacyCalendar.datetime_to_string/1             1  0.00    0    0.00
LegacyCalendar.date_to_string/3                 1  0.00    0    0.00
anonymous fn/0 in CalendarStringBench.run/0     1  0.00    0    0.00
LegacyCalendar.zero_pad/2                       7 16.67    1    0.14
:erlang.iolist_to_binary/1                      1 16.67    1    1.00
:erlang.integer_to_binary/1                     7 16.67    1    0.14
:erlang.apply/2                                 1 50.00    3    3.00

Profile done over 10 matching functions

```

### Final Optimized Implementation

```
Name                     ips        average  deviation         median         99th %
LegacyCalendar        9.15 M      109.30 ns ±12941.66%          80 ns         140 ns

Extended statistics:

Name                   minimum        maximum    sample size                     mode
LegacyCalendar           70 ns    23697039 ns         7.68 M                    80 ns

Memory usage statistics:

Name              Memory usage
LegacyCalendar           432 B

**All measurements for memory usage were the same**

Profiling LegacyCalendar with eprof...

Profile results of #PID<0.546908.0>
#                                           CALLS     % TIME µS/CALL
Total                                          16 100.0    6    0.38
LegacyCalendar.time_to_string_format/3          1  0.00    0    0.00
LegacyCalendar.time_to_string/4                 1  0.00    0    0.00
LegacyCalendar.microseconds_to_iodata/2         1  0.00    0    0.00
LegacyCalendar.datetime_to_string/1             1  0.00    0    0.00
LegacyCalendar.date_to_string/3                 1  0.00    0    0.00
anonymous fn/0 in CalendarStringBench.run/0     1  0.00    0    0.00
:erlang.integer_to_binary/1                     2 16.67    1    0.50
LegacyCalendar.zero_pad/2                       7 33.33    2    0.29
:erlang.apply/2                                 1 50.00    3    3.00

Profile done over 10 matching functions

```

# Final Source Code

Here's the complete optimized version that's similar to Elixir's master branch:

```elixir
defmodule LegacyCalendar do
  def datetime_to_string(dt) do
    %{
      year: year,
      month: month,
      day: day,
      hour: hour,
      minute: minute,
      second: second,
      microsecond: microsecond
    } = dt

    [
      date_to_string(year, month, day),
      ?\s,
      time_to_string(hour, minute, second, microsecond)
      | "Z"
    ]
  end

  defp time_to_string(hour, minute, second, {microsecond, precision}) do
    [
      time_to_string_format(hour, minute, second),
      ?.
      | microseconds_to_iodata(microsecond, precision)
    ]
  end

  def microseconds_to_iodata(_microsecond, 0), do: []
  def microseconds_to_iodata(microsecond, 6), do: zero_pad(microsecond, 6)

  def microseconds_to_iodata(microsecond, precision) do
    num = div(microsecond, div_factor(precision))
    zero_pad(num, precision)
  end

  defp div_factor(1), do: 100_000
  defp div_factor(2), do: 10_000
  defp div_factor(3), do: 1_000
  defp div_factor(4), do: 100
  defp div_factor(5), do: 10

  defp time_to_string_format(hour, minute, second) do
    [zero_pad(hour, 2), ?:, zero_pad(minute, 2), ?: | zero_pad(second, 2)]
  end

  defp date_to_string(year, month, day) do
    [zero_pad(year, 4), ?-, zero_pad(month, 2), ?- | zero_pad(day, 2)]
  end

  defp zero_pad(val, count) when val >= 0 do
    num = Integer.to_string(val)

    case max(count - byte_size(num), 0) do
      0 -> num
      1 -> ["0" | num]
      2 -> ["00" | num]
      3 -> ["000" | num]
      4 -> ["0000" | num]
      5 -> ["00000" | num]
    end
  end

  defp zero_pad(val, count) do
    [?- | zero_pad(-val, count)]
  end
end
```

# Key Differences and Further Optimization Potential

The final version includes several important characteristics:

1. **Returns IOList Instead of String**
   - The function now returns an IOList directly instead of converting to a string
   - This saves memory when used with IOData-aware libraries like Phoenix or JSON encoders
   - No need for final string creation and garbage collection

2. **Improper Lists Throughout**
   - Uses improper lists (`|`) at the end of lists instead of regular list construction
   - Improves performance by reducing list traversal overhead
   - Example: `[zero_pad(hour, 2), ?:, zero_pad(minute, 2), ?: | zero_pad(second, 2)]`

3. **Further Optimization Possibilities**
   - Special case for years 2020-2029:
     ```elixir
     defp zero_pad(twentytwenty, 4) when twentytwenty >= 2020 and twentytwenty < 2030 do
       ones = twentytwenty - 2020
       [?2, ?0, ?2, ones + ?0]
     end
     ```
   - Special case for two-digit numbers (used frequently in time formatting):
     ```elixir
     defp zero_pad(ones, 2) when ones >= 0 and ones < 10 do
       [?0, ones + ?0]
     end

     defp zero_pad(val, 2) when val >= 10 and val < 100 do
       tens = div(val, 10)
       ones = rem(val, 10)
       [tens + ?0, ones + ?0]
     end
     ```
   - These optimizations could further improve performance for common cases

4. **Trade-offs**
   - Adding more special cases increases code complexity
   - The performance gain might not justify the maintenance cost
   - Current implementation strikes a balance between performance and maintainability

# Final Thoughts

The optimization journey took us from a simple string concatenation approach to a highly optimized implementation using BEAM-specific features. The final version in Elixir's master branch represents a careful balance between performance and maintainability.

While there's room for further optimization through special cases, the current implementation already provides excellent performance with reasonable code complexity. The decision to return IOLists directly also provides flexibility for consumers of this API, allowing them to choose between string conversion and direct IOList usage based on their needs. The new JSON module already benefits from this.

## A Note on Performance Measurements

When measuring performance, it's essential to understand the difference between simple timing tools like `:timer.tc` and comprehensive benchmarking frameworks like Benchee. In our initial measurements using `:timer.tc`, we saw around 3,150 microseconds per iteration, while Benchee showed a median of 551 nanoseconds per operation. This significant difference occurs because `:timer.tc` includes the overhead of the `for` loop and doesn't account for VM optimizations that happen during Benchee's warm-up phase.

For accurate performance comparisons, it's crucial to:
1. Use proper benchmarking tools that account for VM warm-up
2. Run multiple iterations to get statistically significant results
3. Consider the median and deviation, not just the average
4. Be aware of the measurement units (microseconds vs nanoseconds)
