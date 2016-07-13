---
layout: post
title: Using Ansible's WinRM configuration script with vSphere guest customization 
---

This is going to be a quick post...

Since this took me a few tries to get this working I thought it would help someone out there if they need Ansible working on boot.  When adding a customization spec add the lines below to `run once`.


```
powershell -command "& {Set-ExecutionPolicy Unrestricted}"
```

```
powershell -command "& {Invoke-Expression ((New-Object System.Net.Webclient).DownloadString(\"https://raw.githubusercontent.com/ansible/ansible/devel/examples/scripts/ConfigureRemotingForAnsible.ps1\"))}"
```

Sources: 
http://www.davidklee.net/2012/09/05/execute-a-powershell-self-elevating-script-after-a-vmware-template-deployment/
http://blog.rolpdog.com/2015/09/manage-stock-windows-amis-with-ansible.html


