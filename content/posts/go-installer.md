---
title: Go installer script
date: 2025-03-16
tags: ["Go"]
author: "Kapil Agrawal"
comments: false
---

I asked our LLM overlords to generate a simple shell script to install Go on a linux system so as to get a consistent development environment regardless of the underlying OS architecture (amd64 vs. arm) I am working on. By default it installs the latest release. To install a specific go version pass it as a cli argument when running the script.

```sh
#!/bin/bash

set -e
set -o pipefail

# Detect architecture
ARCH_RAW=$(uname -m)
case "$ARCH_RAW" in
    x86_64) GO_ARCH="amd64" ;;
    aarch64 | arm64) GO_ARCH="arm64" ;;
    *)
        echo "❌ Unsupported architecture: $ARCH_RAW" >&2
        exit 1
        ;;
esac

# Get version (default: latest)
if [ -n "$1" ]; then
    VERSION="$1"
else
    echo "📦 Fetching latest Go version..."
    VERSION=$(curl -s https://go.dev/VERSION?m=text | head -n 1 | sed 's/go//')
fi

ARCH="linux-${GO_ARCH}"
GO_TARBALL="go${VERSION}.${ARCH}.tar.gz"
GO_URL="https://go.dev/dl/${GO_TARBALL}"
INSTALL_DIR="/usr/local/go"

# Root check
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
  rm -f "$(command -v go)" "$(command -v gofmt)"
fi

# Download and install Go
if wget --quiet --spider "$GO_URL"; then
  echo "⬇️  Downloading Go $VERSION for $GO_ARCH..."
  wget -q "$GO_URL" -O "$GO_TARBALL"
  echo "📦 Extracting Go..."
  tar -C /usr/local -xvf "$GO_TARBALL"
  rm "$GO_TARBALL"
else
  echo "❌ Error: Go version $VERSION not found at $GO_URL" >&2
  exit 1
fi

# Create symlinks
ln -sf /usr/local/go/bin/go /usr/local/bin/go
ln -sf /usr/local/go/bin/gofmt /usr/local/bin/gofmt

echo "✅ Go $VERSION installed successfully for $GO_ARCH!"
echo "👉 Note: GOPATH will default to \$(go env GOPATH) (usually ~/go)."
echo "👉 Add /usr/local/go/bin to your PATH if not already set."
```
