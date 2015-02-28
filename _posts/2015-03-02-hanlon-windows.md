---
layout: post
title: Hanlon now supports Windows provisioning!
---


# WinPE Build Process

We tried to make this as painless as possible.  




{% highlight PowerShell linenos %}

Get-ChildItem env:

{% endhighlight %}

# WinPE Hanlon Static Location

Within your static location three files must be present, boot.wim, BCD and boot.sdi.

{% highlight bash %}

wimboot
sources
└── boot.wim
boot
├── bcd
├── BCD
└── boot.sdi


{% endhighlight %}
