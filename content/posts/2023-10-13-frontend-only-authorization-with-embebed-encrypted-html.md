---
title: Frontend-only authorization with embebed encrypted HTML
description: ""
date: 2023-10-13T03:39:31.264Z
draft: false
tags:
    - cryptography
slug: frontend-authorization-embebed-encrypted-html
keywords:
    - cryptography, frontend
---

I spent the day creating [privaturl](https://github.com/joaquinlpereyra/privaturl), a very cheap 
way of sharing content with certain recipients but hosting it on the public web. No server needed, 
it all[1] works on the frontend and relies on you sharing a link/key with the recipients via a secure 
communication channel (like Signal or Whatsapp).

![meme](/privaturl.jpg)

The encryption is done using `AES-GCM`. It's all pretty standard and all the details are on Github anyway, 
so go check it out.

There's zero chance I was the first one to think or code something like this, but I haven't been able 
to find any other implementation. If you know of one, let me know!

1: Not all. You need a computer to actually encrypt the page once.