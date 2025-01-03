---
title: "Running Livebook as a Systemd Service for Your User"
meta_title: ""
description: "Livebook is a powerful tool for creating and sharing interactive notebooks with Elixir. To make it even more convenient, you can set it up to run as a systemd service..."
date: 2024-06-30T07:32:15Z
image: "/images/image-placeholder.png"
categories: ["Development", "Tutorial"]
author: "Daniel Kukula"
tags: ["elixir", "livebook", "systemd", "linux"]
draft: false
---

Livebook is a powerful tool for creating and sharing interactive notebooks with Elixir. To make it even more convenient, you can set it up to run as a systemd service for your user. This ensures that Livebook starts automatically whenever you log in, and runs in the background without requiring root permissions. Here's a step-by-step guide to help you get started.

## Step 1: Install Livebook

First, ensure you have Livebook installed. You can install it via Elixir's package manager, mix, by running:

```sh
mix escript.install hex livebook
```

## Step 2: Create a User Systemd Service File

Systemd allows users to manage their own services. User-specific service files are stored in `~/.config/systemd/user/`.

```sh
mkdir -p ~/.config/systemd/user
nano ~/.config/systemd/user/livebook.service
```

## Step 3: Define the Service Configuration

Add the following content to the `livebook.service` file (replace USER and PASSWORD):

```ini
[Unit]
Description=Livebook Service

[Service]
Type=simple
ExecStart=/home/__USER__/.asdf/shims/livebook server
Restart=on-failure
Environment=HOME=%h
Environment=MIX_ENV=prod
Environment=PATH=/home/USER/.asdf/shims:/usr/local/bin:/usr/bin:/bin
Environment=LIVEBOOK_HOME=%h/livemd
Environment=LIVEBOOK_PASSWORD=__PASSWORD__
Environment=LIVEBOOK_PORT=8090
Environment=LIVEBOOK_IFRAME_PORT=8091


[Install]
WantedBy=default.target
```

**Explanation**:
- `ExecStart`: The command to start Livebook. Ensure the path to the Livebook binary is correct.
- `Environment`: Sets necessary environment variables. `%h` will expand to the user's home directory.

## Step 4: Reload User Systemd Configuration

After creating the service file, reload the systemd user configuration.

```sh
systemctl --user daemon-reload
```

## Step 5: Enable the Service for Your User

Enable the service to start on user login.

```sh
systemctl --user enable livebook.service
```

## Step 6: Start the Service for Your User

Start the Livebook service.

```sh
systemctl --user start livebook.service
```

## Step 7: Check the Service Status

Verify that the service is running.

```sh
systemctl --user status livebook.service
```

## Step 8: Enable User Systemd Services on Boot

Ensure that user services are started automatically at boot. You can enable lingering for your user:

```sh
sudo loginctl enable-linger USER
```

Replace `USER` with your actual username.

## Conclusion

By following these steps, you can set up Livebook to run as a systemd service for your user. This setup ensures that Livebook starts automatically when you log in and runs smoothly in the background. Enjoy your interactive notebooks with the convenience of a managed service!
