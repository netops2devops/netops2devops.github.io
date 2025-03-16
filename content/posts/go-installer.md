---
title: Go installer script
date: 2025-03-16
tags: ["Go", "Programming"]
author: "Kapil Agrawal"
comments: false
---

I wrote a super simple shell script to install Go on a linux system to get a consistent Go environment for all users. I often use this script when I need Go installed on a remote development VM specialy when working on projects that require x86-64 architecture.

```sh
#!/bin/bash

set -e           # Exit on error
set -o pipefail  # Catch errors in pipelines

VERSION=${1:-"1.24.1"}
ARCH="linux-amd64"
GO_TARBALL="go${VERSION}.${ARCH}.tar.gz"
GO_URL="https://go.dev/dl/${GO_TARBALL}"
INSTALL_DIR="/usr/local/go"

if [ "$(id -u)" -ne 0 ]; then
  echo "This script must be run as root or with sudo." >&2
  exit 1
fi

# Remove existing Go installation
if [ -d "$INSTALL_DIR" ]; then
  echo "Removing existing Go installation..."
  rm -rf "$INSTALL_DIR"
fi

# Remove old Go binaries
if command -v go &>/dev/null; then
  echo "Removing existing Go binaries..."
  rm -f $(which go) $(which gofmt)
fi

# Download and install Go
if wget --quiet --spider "$GO_URL"; then
  echo "Downloading Go $VERSION..."
  wget -q "$GO_URL" -O "$GO_TARBALL"
  echo "Extracting Go..."
  tar -C /usr/local -xvf "$GO_TARBALL"
  rm "$GO_TARBALL"
else
  echo "Error: Go version $VERSION not found at $GO_URL" >&2
  exit 1
fi

# Ensure symbolic links are created
ln -sf /usr/local/go/bin/go /usr/local/bin/go
ln -sf /usr/local/go/bin/gofmt /usr/local/bin/gofmt

# Configure GOPATH
GOPATH="/opt/go"
mkdir -p "$GOPATH/bin" "$GOPATH/pkg"
chown -R root:sudo "$GOPATH"
chmod -R 0775 "$GOPATH"

grep -q "^GOPATH=" /etc/environment || echo "GOPATH=$GOPATH" >> /etc/environment

echo "Go $VERSION installed successfully!"
```
