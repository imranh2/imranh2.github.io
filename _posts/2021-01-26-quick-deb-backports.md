---
layout: post
title: "My quick and dirty notes on building deb backports"
date: 2021-01-26 19:45
last_modified_at: 2022-03-29 18:24
---

1. Grab the sources

`dget http://deb.debian.org/debian/pool/main/h/htop/htop_3.0.5-2.dsc`

2. Install build dependcies

`sudo mk-build-deps --install --remove`

3. Bump the package version

`dch --bpo`

4. Test building ([apt-file](https://wiki.debian.org/apt-file) might help)

`fakeroot debian/rules binary`

5. Build the deb!

`dpkg-buildpackage -b --no-sign`