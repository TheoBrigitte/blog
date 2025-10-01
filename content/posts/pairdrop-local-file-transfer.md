---
title: "Easy Cross-Platform File Transfer with PairDrop"
date: 2025-10-01T12:00:00+01:00
description: "A simple bash script to manage PairDrop for seamless file transfers across devices on your local network"
---

## The Problem

Transferring files between devices on a local network can be surprisingly annoying. You might need to send a file from your phone to your laptop, or from your Linux machine to your friend's Windows PC. While cloud services work, they're overkill for local transfers and require internet connectivity.

## The Solution: PairDrop

[PairDrop](https://github.com/schlagmichdoch/PairDrop) is a self-hosted alternative to Apple's AirDrop that works across platforms. It's a web-based tool that allows devices on the same network to discover each other and transfer files directly through the browser.

## A Simple Management Script

I wrote a bash script to make running PairDrop even easier. Instead of manually managing Docker containers, this script handles everything with simple `start` and `stop` commands.

### Features

The script handles:
- **Automatic startup** of the PairDrop Docker container
- **mDNS detection** to warn if local network discovery might not work
- **Health checking** to ensure PairDrop is actually running before reporting success
- **Clean shutdown** of the container
- **User-friendly output** with colored status messages

### How It Works

The script uses Docker to run PairDrop's LinuxServer.io image and exposes it on port 80. It leverages mDNS (multicast DNS) so devices can reach PairDrop using a `.local` hostname without needing to know the exact IP address.

```bash
# Start PairDrop
pairdrop

# Stop PairDrop
pairdrop stop
```

Once running, any device on your network can visit `http://your-hostname.local` and start transferring files.

### Key Implementation Details

**mDNS Check**: The script verifies mDNS is enabled before starting:
```bash
if ! resolvectl status | grep '+DefaultRoute' | grep '+mDNS' &>/dev/null; then
  echo "[WARN] mDNS is not enabled on this host."
fi
```

**Health Verification**: Rather than assuming Docker started successfully, it polls the service:
```bash
while ! curl -s "http://localhost:$hostport" &>/dev/null; do
  echo "[INFO] Waiting for PairDrop to start..."
  sleep 2
done
```

**Clean Container Management**: The container runs with `--rm` to auto-cleanup, and the script checks if it's already running to avoid duplicates.

## Setup Requirements

To use this script, you'll need:

1. **Docker** installed on your host machine
2. **mDNS enabled** for seamless `.local` hostname resolution (see [Arch Wiki guide](https://wiki.archlinux.org/title/Systemd-resolved#mDNS))
3. The script saved to your `$PATH` (e.g., `~/.local/bin/pairdrop`)

## Why I Like This Approach

This script embodies a few principles I value:

- **Simplicity**: Two commands to start and stop
- **Containerization**: No system pollution, easy cleanup
- **Local-first**: No cloud services, no external dependencies
- **Cross-platform**: Works with any device that has a browser

File transfers shouldn't require thinking about protocols, ports, or IP addresses. With PairDrop and this wrapper script, they don't have to.

## Full Script

You can find the complete script in my dotfiles or check out the [PairDrop project](https://github.com/schlagmichdoch/PairDrop) for more details on hosting your own instance.
