---
layout: post
title: "Using Windows Integrated Authentication with VMware App Volumes"
date: 2019-09-20 12:00
---

When installing App Volumes using a external MS SQL Server you are faced with 
the question of what authentication method to use, Integrated or SQL Server.

Integrated is always preferred however there's a catch, the installer says it'll
use the server's SYSTEM account, so that means something like `DOMAIN\SERVER$`.

In SQL Server you can't make a login for a Computer Object, so you have to:

* Create a Security Group
* Add the server(s) to the Security Group
* Create a Login for the Security Group and assign it permission on the database
