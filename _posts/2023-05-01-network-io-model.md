---
layout: post
title:  "网络IO模型的个人梳理"
date:   2023-05-01 +0800
categories: network reactor
---

# 网络IO模型的个人梳理

本文对这些模型的梳理仅代表个人的看法，按照本人的思路从最简单的单线程阻塞IO模型如何扩展进化到Reactor及Proactor的梳理。

![从单线程阻塞IO模型到Proactor](../image/network-io-model.png)