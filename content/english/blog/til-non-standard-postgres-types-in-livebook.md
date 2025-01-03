---
title: "TIL: Non Standard Postgres Types in Livebook"
meta_title: ""
description: "Livebook has a new, nice feature that allows you to connect to a database. It works with basic data types..."
date: 2023-01-08T13:01:43Z
image: "https://dev-to-uploads.s3.amazonaws.com/uploads/articles/jdeansqhyq5ey9647oo3.png"
categories: ["Development", "Tips"]
author: "Daniel Kukula"
tags: ["postgresql", "livebook", "elixir"]
draft: false
---

Livebook has a new, nice feature that allows you to connect to a database. It works with basic data types.

![livebook screenshot](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/jdeansqhyq5ey9647oo3.png)

But I found a problem when the data type is not supported by postgrex.
![livebook screenshot](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/vvu21nwb2ba0pdfept1c.png)

In this case, you can either have to write a custom postgrex extension like the example [here](https://github.com/elixir-ecto/postgrex/issues/439). In postgrex GitHub repo, there are many [examples](https://github.com/elixir-ecto/postgrex/tree/master/lib/postgrex/extensions) for common data types, but XML is missing there.

You can also cast your field to a supported data type:
```sql
SELECT demographics::text from person.person   
```

![livebook screenshot](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/51c7kecpqws2dmqlayut.png)

To play with this example, you can install the example [Adventureworks](https://learn.microsoft.com/en-us/sql/samples/adventureworks-install-configure) database provided by Microsoft. I made an [elixir converter](https://github.com/dkuku/AdventureWorks-for-Postgres) that converts some of the files to a format accepted by PostgreSQL, and also updated the previous version to properly name the columns with underscores.
