---
layout: post
title: "Managing dynamic ip sets in firewall rules with iptables and ipset"
date: 2012-09-24 15:23
comments: true
categories: [Linux, iptables]
---

There was a need to configure particular server to apply a set of iptables rules to dynamically comprised set of ip addresses. Trying to solve this problem using only plain old iptables rules can quickly become a mess, especially considering that large number of rules drastically diminishes performance of machine in general, set aside becoming a pain adding/removing entries from the list. Fortunately, there is a tool out there allowing us to solve this problem in a neat and tidy way, and it is called [ipset](http://ipset.netfilter.org/).
<!--more-->

## Problem statement

