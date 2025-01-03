---
title: "Phoenix Live Dashboard Custom Page"
meta_title: ""
description: "Today I played with custom live dashboard pages, That's a short documentation how to use it."
date: 2020-12-25T11:15:31Z
image: "https://dev-to-uploads.s3.amazonaws.com/i/mn8nd2tsd6iqz26ia0hf.png"
categories: ["Development"]
author: "Daniel Kukula"
tags: ["elixir"]
draft: false
---

Today I played with custom live dashboard pages, That's a short documentation how to use it. We will implement a page that shows the currently running system processes on the box running our elixir system.

Documentation can be found on [hexdocs](https://hexdocs.pm/phoenix_live_dashboard/Phoenix.LiveDashboard.PageBuilder.html).
Most up to date code is attached in the gist.

First we need to create a new module:

```elixir
defmodule Phoenix.LiveDashboard.OsProcesses do
  @moduledoc false
  use Phoenix.LiveDashboard.PageBuilder

end
```

To get the list I'll use the output of `ps aux` command.
I'll cheat here a bit and use `jq` to convert the page to json which later can be easily decoded by Jason. This also shows how to use more complicated commands. We can of course parse the string in elixir.

To use pipes inside the command I had to use the os module with a character list passed as an option. In the gist the `ps` parser is written in pure elixir but for simplicity I'll leave the `jq` version here. On windows you can play with `tasklist` the output is a bit similar but there are spaces on the first line so you may want to parse it a bit different.

```elixir
  defp fetch_processes(_params, _node) do
    data = 
    :os.cmd('ps aux | jq -sR \'[sub("\n$";"") | splits("\n") | sub("^ +";"") | [splits(" +")]] | .[0] as $header | .[1:] | [.[] | [. as $x | range($header | length) | {"key": $header[.], "value": $x[.]}] | from_entries]\'')
    |> Jason.decode!(%{keys: :atoms!})

    {data, length(data)}
  end
```

The return value of this function must be a tuple with rows and the length of the rows.
Next function we need to write is the columns definitions:

```elixir
  defp columns() do
    [
      %{
        field: :"PID",
        sortable: :asc
      },
      %{
        field: :"TTY"
      },
      %{
        field: :"TIME",
        cell_attrs: [class: "text-right"],
      },
      %{
        field: :"USER",
      },
      %{
        field: :"%CPU",
        sortable: :desc
      },
      %{
        field: :"%MEM",
        sortable: :desc
      },
      %{
        field: :"VSZ",
        sortable: :asc
      },
      %{
        field: :"RSS",
        sortable: :asc
      },
      %{
        field: :"STAT",
      },
      %{
        field: :"START",
        sortable: :asc
      },
      %{
        field: :"COMMAND",
      },
    ]
  end
```

Both of this functions need to be passed to the render_page/1 callback:

```elixir
  @impl true
  def render_page(_assigns) do
    table(
      columns: columns(),
      id: :os_processes,
      row_attrs: &row_attrs/1,
      row_fetcher: &fetch_processes/2,
      rows_name: "tables",
      title: "OS processes",
      sort_by: :"PID"
    )
  end
```

As we see there is another function required, the row_attrs that returns html attributes for our table rows.
And thats basically it.

Last two pieces of code is the menu_link/2 callback:

```elixir
  @impl true
  def menu_link(_, _) do
    {:ok, "OS Processes"}
  end
```

and the router entry for our page:

```elixir
    scope "/" do
      pipe_through :browser
    live_dashboard "/dashboard", metrics: ChatterWeb.Telemetry,
    additional_pages: [
    os_processes: Phoenix.LiveDashboard.OsProcesses
  ]
```

And that's basically it - The code can be found on github, with the params used for sorting, limiting and search. Have fun and link any pages you have created.
