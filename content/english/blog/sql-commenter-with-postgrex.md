---
title: "SQL Commenter with Postgrex"
meta_title: ""
description: "A few years ago, I discovered sqlcommenter, a tool that enables adding trace context to SQL queries..."
date: 2024-12-15T21:24:29Z
image: "/images/image-placeholder.png"
categories: ["Development", "Tutorial"]
author: "Daniel Kukula"
tags: ["elixir", "postgresql", "ecto", "postgrex"]
draft: false
---

A few years ago, I discovered sqlcommenter, a tool that enables adding trace context to SQL queries. This feature makes it possible to match database calls visible in PostgreSQL logs or any observability tool. At the time, I found several questions about this topic scattered across the internet, but no concrete solutions. I attempted to solve this myself and [experimented with various approaches](https://dev.to/dkuku/enhancing-sql-traceability-with-sqlcommenter-in-elixir-5d9f), but none of them proved viable for implementation in Elixir, particularly with Ecto.

Since then, I've revisited this idea several times. Recently, I've taken a deeper dive into the subject and gained a better understanding of how the Ecto machinery works under the hood.

I created a pull request that added the ability to include comments in Postgrex queries, which was released in version 0.19.3. Now you can add `comment: "this is the comment that will be added to query"`, and it will appear in the logs like this:

```
2024-12-15 20:02:33.328 GMT,"postgres",
"connector_test",29538,"127.0.0.1:42156",675f35d4.7362,57,
"SELECT",2024-12-15 20:02:28 GMT,33/50,240124,LOG,00000,
"execute <unnamed>: SELECT c0.""id"" FROM ""conversation"" AS c0
 WHERE (c0.""id"" = $1)/*this is the comment that will be added to query*/","parameters: $1 = '0193cbea-58ee-7f1c-8250-dd25c8e9f6cf'",
,,,,,,,,"","client backend",,0
```

To add comments to every query, we can use Ecto.Repo's callback functions. Specifically, the `prepare_query/3` callback:

```elixir
@callback prepare_query(operation, query :: Ecto.Query.t(), opts :: Keyword.t()) ::
  {Ecto.Query.t(), Keyword.t()}
```

Where `operation` can be `:all | :update_all | :delete_all | :stream | :insert_all`.

We can use this callback to modify both Ecto and Postgrex options and add comments:

```elixir
def prepare_query(_operation, query, opts) do
  comment = "comment that will be appended"
  {query, [comment: comment, prepare: :unnamed] ++ opts}
end
```

This is the basic implementation, but you can enhance it by copying the telemetry context. Note the `prepare: :unnamed` option - this disables prepared statements. Adding a comment always disables prepared statements, but I included it here explicitly for clarity. This is due that PostgreSQL uses the actual query string for prepared statements, but there are some important nuances to understand:

1. Query String as a Key: In PostgreSQL, the prepared statement is associated with a specific query string. This means if you prepare a query like SELECT * FROM table WHERE id = $1, PostgreSQL uses that exact query string as a reference.
2. Query Parsing: When a prepared statement is created, PostgreSQL parses the query string to generate a query plan. This query plan is then stored and reused when you execute the statement.
3. String Variations Matter: Small variations in the query string, such as extra spaces, case changes, or different formatting, result in PostgreSQL treating them as different queries. For example:

`SELECT * FROM table WHERE id = $1/*comment1*/`
`SELECT * FROM table WHERE id = $1/*comment2*/`

These would be treated as distinct prepared statements due to their string differences.

Sqlcommenter requires a [specific format](https://google.github.io/sqlcommenter/spec/) of the message. 
The sqlcommenter format follows a specific convention where each key-value pair is URL-encoded and joined with commas. For example, `app='connector',caller='MyModule.function'` becomes `app='connector',caller='MyModule%2Efunction'`. This standardized format ensures consistency across different programming languages and frameworks while making the comments both human-readable and parseable by observability tools. Special characters and spaces are encoded to prevent SQL injection and maintain valid query syntax.
I created previously a [package](https://hex.pm/packages/sqlcommenter) that you can use to encode your queries. 
Full repo module that uses this package looks like this:

```elixir
defmodule Connector.Repo do
  use Ecto.Repo,
    otp_app: :connector,
    adapter: Ecto.Adapters.Postgres

  @impl true
  def default_options(_operation) do
    [stacktrace: true]
  end

  @impl true
  def prepare_query(_operation, query, opts) do
    caller = Sqlcommenter.extract_repo_caller(opts, __MODULE__)
    node = if node() == :nonode@nohost, do: "connector", else: Atom.to_string(node())

    sqlcommenter = [
      app: "connector",
      caller: caller,
      node: node
    ]

    comment = Sqlcommenter.to_str(sqlcommenter)
    {query, [comment: comment, prepare: :unnamed] ++ opts}
  end
end
```

The `default_options/1` is another Ecto.Repo callback that allows you to inject options at an earlier stage. Using this callback gives us access to the stacktrace in the options passed to `prepare_query`, which I used to identify where our query was called from. Another thing I added here is the node name.
Here's how it appears in the PostgreSQL log (I added few newlines for visibility):

```
2024-12-15 20:48:19.541 GMT,"postgres",
"connector_test",35894,"127.0.0.1:52118",675f408e.8c36,96,
"SELECT",2024-12-15 20:48:14 GMT,21/61,240492,LOG,00000,
"execute <unnamed>: 
SELECT m0.""id"", m0.""role"" AS m0 WHERE (m0.""id"" = $1)
/*app='connector',caller='Connector.ChatTest.test%20messages%20get_message%21%2F1%20returns%20the%20message%20with%20given%20id%2F1',node='connector'*/",
"parameters: $1 ='0193cc14-4054-7dd5aaad-5943c073e8e6'",
,,,,,,,,"","client backend",,0  
```

I hope you found this information useful and that it adds another tool to your debugging toolkit. For cases where the comment remains static, like the one mentioned above, I'm currently working on another example that uses an experimental Ecto feature. If you're interested in learning more, please leave a like to let me know that you find this content valuable.
I wrapped this also as a [package](https://github.com/dkuku/opentelemetry_sqlcommenter) - contribute when you have ideas how to extend it.
