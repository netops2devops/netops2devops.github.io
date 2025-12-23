---
title: Go installer script
date: 2025-03-16
tags: ["Go"]
author: "Kapil Agrawal"
comments: false
---

Say what you may about Shell scripts but fact of the matter is that a shell script written 20 years ago still works the same way as it did back then without requiring any external dependencies or special tool chain to get reproducible results.

I asked our LLM overlords to generate a simple shell script to install Go on a linux system so to get a consistent development environment regardless of the underlying OS architecture (amd64 vs. arm). By default it installs the latest stable release. To install a specific go version pass it as a cli argument when running the script.

{{< gist "netops2devops" "35f4a8dad78704fed04fd3ba0261f9c1" >}}
