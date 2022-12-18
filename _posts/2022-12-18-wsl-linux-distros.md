---
layout: post
title: "Easily Importing distros in to WSL"
date: 2022-12-18 19:20
last_modified_at: 2022-12-18 19:20
---

Did you know you can take a docker container and WSL it?

We won't go over [installing WSL](https://learn.microsoft.com/en-us/windows/wsl/install) here, but either 1 or 2 works.

# Quick How To

Lets say we want debian bookworm

0. `export CONTAINER='debian:bookworm'`
1. `podman pull $CONTAINER`
2. `export CONTAINER_ID=$(podman container ls -a | grep -i $CONTAINER | awk '{print$1}')`
3. `podman export $CONTAINER_ID > /mnt/c/Users/$USER/$(echo $CONTAINER | sed s/:/-/).tar`

Then from Windows/Powershell

4. `wsl.exe --import --version 2 debian-bookworm 'C:\Users\$USER\debian-bookworm' 'C:\Users\$USER\debian-bookworm.tar'`

```
First arg is the distro name

Second arg is the folder in which to store the distro

Third arg is the source .tar file for the distro
```

# Setting a different User

```bash
export NEW_USER=imranh
useradd -m -G sudo -s /bin/bash "$NEW_USER"
passwd "$NEW_USER"
```

```bash
echo -e "[user]\ndefault=$NEW_USER" >> /etc/wsl.conf
```


# Useful WSL Commands

## List all WSL Instances/Distros

```Powershell
wsl --list --all --verbose
```

## Shutdown an instance

```Powershell
wsl --shutdown <name>
```

OR

```Powershell
wsl --terminate <name>
```

## Delete WSL Instance

```Powershell
wsl --unregister <name>
```