---
layout: post
title: "My quick and dirty notes on building deb backports"
date: 2021-01-26 19:45
last_modified_at: 2022-08-27 11:37
---

Tired of Googling and not doing this to enough to remember so here's my notes:

1. Grab the sources
```bash
dget http://deb.debian.org/debian/pool/main/h/htop/htop_3.0.5-2.dsc
```
2. Install build dependcies
```bash
sudo mk-build-deps --install --remove
```
3. Bump the package version
```bash
dch --bpo
```
4. Test building ([apt-file](https://wiki.debian.org/apt-file) might help)
```bash
fakeroot debian/rules binary
```
5. Build the deb!
```bash
dpkg-buildpackage -b --no-sign
```