---
title: "Nginx Logs in Live Dashboard"
meta_title: ""
description: "Building on top of my previous blog posts on creating a nginx log parser and displaying ps output in..."
date: 2020-12-30T14:51:47Z
image: "/images/image-placeholder.png"
categories: ["Development", "Tutorials"]
author: "Daniel Kukula"
tags: ["elixir", "nginx", "phoenix"]
draft: false
---

Building on top of my previous blog posts on creating a [nginx log parser](https://dev.to/dkuku/parse-nginx-access-log-with-nimbleparsec-e1l) and displaying [ps output in live dashboard](https://dev.to/dkuku/phoenix-live-dashboard-custom-page-4chj) I decided to build a nginx log display.

A problem that may occur is that the log file may be huge and we don't want to load and parse it all. I decided to load just the end of the file, check if we got enough data and if not then repeat until we load enough.

```elixir
  def fetch_logs(params, node) do
    data = :rpc.call(node, __MODULE__, :logs_callback, [params])

    {data, length(data)}
  end
```

Our function call is wrapped in a rpc.call so it's compatible with live view multiple nodes. We also pass the sort and search params so the filtering can be done on the node with the log files - send the functions to the data.

Our callback opens the file, gets the file size and passes it down to a recursive function which handles the file reading. Then after getting the data back we sort it and return only the required amount of rows.

```elixir
  def logs_callback(params) do
    %{limit: limit, sort_by: sort_by, sort_dir: sort_dir} = params

    with {:ok, pid} <- :file.open(@path, [:binary]),
         {:ok, info} <- :file.read_file_info(pid),
         {:file_info, file_size, _, _, _, _, _, _, _, _, _, _, _, _} <- info,
         {:ok, content} <- get_data([], pid, file_size, 0, params),
         :ok <- :file.close(pid) do
           content
           |> Enum.sort_by(&sort_value(&1, sort_by), sort_dir)
           |> Enum.take(limit)
    else
      error -> IO.puts(error)
    end
  end
```

The recursive function reads some amount of data from the end of the file, splits it to separate rows remembers the first one which most of the time is not full row so we just remember it for now. The rest is filtered and parsed. Then we check if we got enough data, if yes we just return it to be displayed on the front end. Otherwise we pass our current parsed lines and the part that was not parsed before to a recursive call where we get another chunk of data from the end of the file we just get it larger by the byte size of the string so the last line does not end in some random place and won't fail in the parser. Then we just repeat the steps. When its done we again check if we got enough data. This is repeated until we reach the limit or we read all of the file content. 

Depending on your web app your @avg_line_length may be different - it's used to calculate how many bytes we want to get from the end of file each time.   

```elixir
@avg_line_lenght 200
  def get_data(data, _pid, 0 = _load_from, _head_size, _params), do: {:ok, data}

  def get_data(data, pid, load_from, last_line_offset, params) do
    %{limit: limit, search: search} = params
    chunk_size = limit * @avg_line_lenght

    {load_from, buffer_size} =
      if load_from - chunk_size < 0 do
        {0, load_from + last_line_offset}
      else
        {load_from - chunk_size, chunk_size + last_line_offset}
      end

    [first_line | full_lines_chunk] = get_data_chunk(pid, load_from, buffer_size)

    updated_data = parse_chunk(full_lines_chunk, search) ++ data

    newlines = Enum.count(updated_data)

    if newlines < limit do
      get_data(updated_data, pid, load_from, byte_size(first_line), params)
    else
      {:ok, updated_data}
    end
  end

  defp get_data_chunk(pid, load_from, buffer_size) do
    case :file.pread(pid, [{load_from, buffer_size}]) do
      {:ok, [:eof]} -> ""
      {:ok, [content]} -> content
      _ -> ""
    end
    |> :binary.split(<<"\n">>, [:global])
  end

  defp parse_chunk(data, search) do
    data
    |> filter_rows(search)
    |> Stream.map(&String.trim/1)
    |> Stream.map(&parse/1)
    |> Stream.filter(fn parsed -> parsed |> elem(0) == :ok end)
    |> Enum.map(&parsed_line_to_map/1)
  end
```

And that's the basic idea. 
Working files can be found on [github](https://gist.github.com/dkuku/8f7e3fbfadb28430cc52733baea40645).
