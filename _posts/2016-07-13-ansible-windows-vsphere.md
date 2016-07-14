---
layout: post
title: Using Ansible's WinRM configuration script with vSphere
---

#### Method #1 - Guest Customization
Since this took me a few tries to get this working I thought it would help someone out there if they need Ansible working on boot.  When adding a customization spec add the lines below to `run once`.


```
powershell -command "& {Set-ExecutionPolicy Unrestricted}"
```

```
powershell -command "& {Invoke-Expression ((New-Object System.Net.Webclient).DownloadString(\"https://raw.githubusercontent.com/ansible/ansible/devel/examples/scripts/ConfigureRemotingForAnsible.ps1\"))}"
```

#### Method #2 - VMware Tools

With VMware Tools installed the configuration of WinRM can be executed via the guest operations API.  There is already an Ansible module - [vmware_vm_shell](https://docs.ansible.com/ansible/vmware_vm_shell_module.html) available for us.  The [playbook](https://github.com/jcpowermac/ansible-vsphere-winrm/blob/master/site.yml) is available below.


```yaml
---

- name: Ansible / WinRM / VMware vSphere Guest Tools 
  hosts: windows
  gather_facts: False
  tasks:
    - name: Enable WinRM to be used with Ansible 
      local_action:
        module: vmware_vm_shell
        hostname: 10.53.252.111
        username: Administrator@vsphere.local
        password: "{{ vsphere_password }}" 
        vm_username: Administrator
        vm_password: "{{ vm_password }}"
        vm_id: ansible-tools
        vm_shell: 'c:\windows\system32\windowspowershell\v1.0\powershell.exe'
        vm_shell_args: "{{ item }}" 
      with_items:
        - '-command "& {Set-ExecutionPolicy Unrestricted}"'
        - '-command "& {Invoke-Expression ((New-Object System.Net.Webclient).DownloadString(\"https://raw.githubusercontent.com/ansible/ansible/devel/examples/scripts/ConfigureRemotingForAnsible.ps1\"))}"'
        - '-command "& {Set-ExecutionPolicy RemoteSigned}"'
```


##### Sources

- http://www.davidklee.net/2012/09/05/execute-a-powershell-self-elevating-script-after-a-vmware-template-deployment/
- http://blog.rolpdog.com/2015/09/manage-stock-windows-amis-with-ansible.html


