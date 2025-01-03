---
title: "Porting Files Generated by Phoenix to Surface"
meta_title: ""
description: "This post is intended to get you started with surface provided components. I provided the original code and surface versions..."
date: 2021-10-27T19:26:31Z
image: "/images/image-placeholder.png"
categories: ["Development", "Tutorials"]
author: "Daniel Kukula"
tags: ["elixir", "phoenix", "surface"]
draft: false
---

This post is intended to get you started with [surface](https://surface-ui.org) provided components. I provided the original code and surface versions so you can compare the differences yourself without installing anything.

After installing surface following the installation guide [https://surface-ui.org/getting_started](https://surface-ui.org/getting_started)
add surface_bulma in your mix.exs, this will allow you to use the table component.

```elixir
{:surface_bulma, "~> 0.2.0"},
{:surface, "~> 0.6.0"},
```

Now add new context for our post:
`mix phx.gen.live Posts Post post title:string body:string`

This will generate a bunch of files in `lib/my_app_web/live/post_live` which we will convert to `surface` versions.
Let's start with adding some imports in `index.ex`.
Change the line:
```elixir
  #use MyAppWeb, :live_view
```

to the following code:
```elixir
  use Surface.LiveView

  alias MyAppWeb.Router.Helpers, as: Routes
  alias SurfaceBulma.Table
  alias SurfaceBulma.Table.Column
  alias Surface.Components.{LivePatch, Link, LiveRedirect}
```

Now rename the `index.html.heex` to `index.sface` and replace the code:
```elixir
<h1>Listing Post</h1>

<%= if @live_action in [:new, :edit] do %>
  <%= live_modal MyAppWeb.PostLive.FormComponent,
    id: @post.id || :new,
    title: @page_title,
    action: @live_action,
    post: @post,
    return_to: Routes.post_index_path(@socket, :index) %>
<% end %>

<table>
  <thead>
    <tr>
      <th>Title</th>
      <th>Body</th>
      <th>Links</th>
    </tr>
  </thead>
  <tbody id="post">
    <%= for post <- @post_collection do %>
      <tr id={"post-#{post.id}"}>
        <td><%= post.title %></td>
        <td><%= post.body %></td>
        <td>
          <span><%= live_redirect "Show", to: Routes.post_show_path(@socket, :show, post) %></span>
          <span><%= live_patch "Edit", to: Routes.post_index_path(@socket, :edit, post) %></span>
          <span><%= link "Delete", to: "#", phx_click: "delete", phx_value_id: post.id, data: [confirm: "Are you sure?"] %></span>
        </td>
      </tr>
    <% end %>
  </tbody>
</table>
<span><%= live_patch "New Post", to: Routes.post_index_path(@socket, :new) %></span>
```

with this content. It's the same code but it uses surface table component:
```elixir
<h1>Listing Post</h1>

{#if @live_action in [:new, :edit]}
  {MyAppWeb.LiveHelpers.live_modal MyAppWeb.PostLive.FormComponent,
    id: @post.id || :new,
    title: @page_title,
    action: @live_action,
    post: @post,
    return_to: Routes.post_index_path(@socket, :index)}
{/if}

<Table data={post <- @post_collection} id={:table} bordered>
  <Column label="Title">
    {post.title}
  </Column>
  <Column label="Body">
    {post.body}
  </Column>
  <Column label="Links">
    <span><LiveRedirect to={Routes.post_show_path(@socket, :show, post)}>Show</LiveRedirect></span>
    <span><LivePatch to={Routes.post_index_path(@socket, :edit, post)}>Edit</LivePatch></span>
    <span><Link click="delete" to="#" values={id: post.id} opts={data: [confirm: "Are you sure?"]}>Delete</Link> </span>
  </Column>
</Table>
<span><LivePatch to={Routes.post_index_path(@socket, :new)}>New Post</LivePatch></span>
```

We will follow the same steps in `show.ex`:
```elixir
  use Surface.LiveView

  alias MyApp.Posts
  alias MyAppWeb.Router.Helpers, as: Routes
  alias Surface.Components.{LivePatch, LiveRedirect}
```

Original code looks like that - we need to rename the file and use our new version:
```elixir
<h1>Show Post</h1>

<%= if @live_action in [:edit] do %>
  <%= live_modal MyAppWeb.PostLive.FormComponent,
    id: @post.id,
    title: @page_title,
    action: @live_action,
    post: @post,
    return_to: Routes.post_show_path(@socket, :show, @post) %>
<% end %>

<ul>
  <li>
    <strong>Title:</strong>
    <%= @post.title %>
  </li>
  <li>
    <strong>Body:</strong>
    <%= @post.body %>
  </li>
</ul>

<span><%= live_patch "Edit", to: Routes.post_show_path(@socket, :edit, @post), class: "button" %></span> |
<span><%= live_redirect "Back", to: Routes.post_index_path(@socket, :index) %></span>
```

`show.sface` content:
```elixir
<h1>Show Post</h1>

{#if @live_action in [:edit]}
  {MyAppWeb.LiveHelpers.live_modal MyAppWeb.PostLive.FormComponent,
    id: @post.id,
    title: @page_title,
    action: @live_action,
    post: @post,
    return_to: Routes.post_show_path(@socket, :show, @post)}
    {/if}

<ul>
  <li>
    <strong>Title:</strong>
    {@post.title}
  </li>
  <li>
    <strong>Body:</strong>
    {@post.body}
  </li>
</ul>

<span><LivePatch to={Routes.post_show_path(@socket, :edit, @post)}, class="button">Edit</LivePatch></span>
<span><LiveRedirect to={Routes.post_index_path(@socket, :index)}>Back</LiveRedirect></span>
```

Last component in this directory is the `form_component.ex` where we need to add:
```elixir
  use Surface.LiveComponent

  alias MyApp.Posts
  alias Surface.Components.Form
  alias Surface.Components.Form.{Field, Label, TextInput, Submit}
```

The template for this component:
```elixir
<div>
  <h2><%= @title %></h2>

  <.form
    let={f}
    for={@changeset}
    id="post-form"
    phx-target={@myself}
    phx-change="validate"
    phx-submit="save">

    <%= label f, :title %>
    <%= text_input f, :title %>
    <%= error_tag f, :title %>

    <%= label f, :body %>
    <%= textarea f, :body %>
    <%= error_tag f, :body %>

    <div>
      <%= submit "Save", phx_disable_with: "Saving..." %>
    </div>
  </.form>
</div>
```

It needs to be replaced with a `form_component.sface` with this code:
```elixir
<div>
  <h2>{@title}</h2>
  <Form for={@changeset} change="validate" submit="save" opts={autocomplete: "off"}>
    <Field name={:title}>
      <Label/>
      <div class="control">
        <TextInput value={@post.title}/>
      </div>
    </Field>
    <Field name={:body}>
      <Label/>
      <div class="control">
        <TextInput value={@post.body}/>
      </div>
    </Field>
    <Submit>Save</Submit>
  </Form>
</div>
```

Last thing that we can replace with surface version is the `modal_component.ex` which you can find in the parent directory.
```elixir
defmodule MyAppWeb.ModalComponent do
  use MyAppWeb, :live_component

  @impl true
  def render(assigns) do
    ~H"""
    <div
      id={@id}
      class="phx-modal"
      phx-capture-click="close"
      phx-window-keydown="close"
      phx-key="escape"
      phx-target={@myself}
      phx-page-loading>

      <div class="phx-modal-content">
        <%= live_patch raw("&times;"), to: @return_to, class: "phx-modal-close" %>
        <%= live_component @component, @opts %>
      </div>
    </div>
    """
  end

  @impl true
  def handle_event("close", _, socket) do
    {:noreply, push_patch(socket, to: socket.assigns.return_to)}
  end
end
```

The surface version looks like that:
```elixir
defmodule MyAppWeb.ModalComponent do
  use Surface.LiveComponent
  alias Surface.Components.{LivePatch, Raw}

  data return_to, :string
  data component, :fun
  data opts, :keyword

  @impl true
  def render(assigns) do
    ~F"""
    <div
      id={@id}
      class="phx-modal"
      phx-capture-click="close"
      phx-window-keydown="close"
      phx-key="escape"
      phx-target={@myself}
      phx-page-loading>

      <div class="phx-modal-content">
        <LivePatch to={@return_to} class="phx-modal-close">
          <#Raw>
            &times;
          </#Raw>
        </LivePatch>
        {live_component @component, @opts}
      </div>
    </div>
    """
  end

  @impl true
  def handle_event("close", _, socket) do
    {:noreply, push_patch(socket, to: socket.assigns.return_to)}
  end
end
```

Surface provides also replacements for [`phx-[event]`](https://surface-ui.org/events) but I had some problems to set it up.
At this point your app should still be functional but using surface components instead of live view provided ones.