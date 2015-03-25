---
layout: post
title: Where ESXi belongs - Nested under KVM
---

I can hear it now, why would you want to do that?  Well for me the motivation was I need to test vSphere (ESXi and vCenter) automation and I don't want to install VMware Workstation to do it.

## First up, Nested ESXi
As always dependencies are first.  I am using Fedora 21 so the packages commands will be related.

{% highlight bash %}
yum groupinstall "Development Tools" -y
yum install gcc-c++ glib2-devel zlibrary zlib-devel pixman-devel libfdt-devel spice-protocol SDL-devel spice-server-devel -y
{% endhighlight %}

### QEMU Changes
Fortunately we have some [very smart people at VMware](https://communities.vmware.com/people/jmattson) that are willing to help our crazy cause.
QEMU has changed since this [post](https://communities.vmware.com/message/2292878#2292878) was originally written and so we must modify the patch.
Below are the modifications required for QEMU 2.3, which is the latest development release - your mileage my vary kinda a thing but I haven't had a problem yet.

{% highlight diff %}
diff --git a/hw/i386/pc_piix.c b/hw/i386/pc_piix.c
index 36c69d7..6952bb3 100644
--- a/hw/i386/pc_piix.c
+++ b/hw/i386/pc_piix.c
@@ -242,8 +242,7 @@ static void pc_init1(MachineState *machine,
     }

     /* init basic PC hardware */
-    pc_basic_device_init(isa_bus, gsi, &rtc_state, &floppy,
-                         (pc_machine->vmport != ON_OFF_AUTO_ON), 0x4);
+    pc_basic_device_init(isa_bus, gsi, &rtc_state, &floppy,TRUE, 0x4);

     pc_nic_init(isa_bus, pci_bus);
{% endhighlight %}

With the modified QEMU lets get the source, patch and recompile.

### Patch and compile QEMU
{% highlight bash %}
cd /opt
sudo git clone https://github.com/qemu/qemu.git
cd qemu
sudo curl "https://gist.githubusercontent.com/jcpowermac/3d9c732be08404302083/raw/ba97ceceefb2ffb085fa8da0f5f5a6142127454e/qemu.patch" | sudo patch -p1
sudo ./configure --enable-kvm --target-list=x86_64-linux-user,x86_64-softmmu
sudo make -j8
sudo wget "https://gist.githubusercontent.com/jcpowermac/36bfa62cd60781264b3f/raw/f26aa286d5ab85f17555141e04ab549e10727475/qemu-kvm"
{% endhighlight %}

This will leave our original QEMU install untouched, which is probably a good thing.  Next up we need to define a virtual machine.

### Create the ESXi virtual machine

There is an issue with ESXi 5.5 and the e1000 network adapter, so we have no real choice except to use vmxnet3.
Below is an example domain specifically for 5.5 because of the inclusion of vmxnet3 adapter.
ESXi 6.0 can and [should](https://communities.vmware.com/message/2446952#2446952) be using e1000.

{% highlight xml %}
<domain type='kvm' id='12'>
  <name>esxi</name>
  <memory unit='KiB'>4194304</memory>
  <currentMemory unit='KiB'>4194304</currentMemory>
  <vcpu placement='static'>2</vcpu>
  <resource>
    <partition>/machine</partition>
  </resource>
  <os>
    <type arch='x86_64' machine='pc-i440fx-2.1'>hvm</type>
  </os>
  <features>
    <acpi/>
    <apic/>
    <pae/>
  </features>
  <clock offset='utc'>
    <timer name='rtc' tickpolicy='catchup'/>
    <timer name='pit' tickpolicy='delay'/>
    <timer name='hpet' present='no'/>
  </clock>
  <on_poweroff>destroy</on_poweroff>
  <on_reboot>restart</on_reboot>
  <on_crash>restart</on_crash>
  <pm>
    <suspend-to-mem enabled='no'/>
    <suspend-to-disk enabled='no'/>
  </pm>
  <devices>
    <emulator>/opt/qemu/x86_64-softmmu/qemu-kvm</emulator>
    <disk type='block' device='disk'>
      <driver name='qemu' type='raw' cache='none' io='native'/>
      <source dev='/dev/virtualmachine/esxi2'/>
      <backingStore/>
      <target dev='sda' bus='sata'/>
      <boot order='2'/>
      <alias name='sata0-0-0'/>
      <address type='drive' controller='0' bus='0' target='0' unit='0'/>
    </disk>
    <controller type='pci' index='0' model='pci-root'>
      <alias name='pci.0'/>
    </controller>
    <controller type='sata' index='0'>
      <alias name='sata0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x06' function='0x0'/>
    </controller>
    <interface type='bridge'>
      <source bridge='ovsbr0'/>
      <vlan>
        <tag id='252'/>
      </vlan>
      <virtualport type='openvswitch'>
      </virtualport>
      <target dev='vnet1'/>
      <model type='vmxnet3'/>
      <alias name='net0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
    </interface>
    <channel type='spicevmc'>
      <target type='virtio' name='com.redhat.spice.0'/>
      <alias name='channel0'/>
      <address type='virtio-serial' controller='0' bus='0' port='1'/>
    </channel>
    <input type='mouse' bus='ps2'/>
    <input type='keyboard' bus='ps2'/>
    <graphics type='spice' port='5901' autoport='yes' listen='127.0.0.1'>
      <listen type='address' address='127.0.0.1'/>
    </graphics>
    <video>
      <model type='vga' vram='9216' heads='1'/>
      <alias name='video0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x0'/>
    </video>
  </devices>
</domain>
{% endhighlight %}


To define a new guest, do the following.  A cd-rom will need to be added and most likely source bridge will need to be changed but the general hardware will be correct.
Any of those changes can be done in virt-manager.
{% highlight bash %}
wget "https://gist.githubusercontent.com/jcpowermac/05e2faa322427f28b907/raw/6149bcc2e5420d13afa96d0f58abc91d6dc84058/esxi.xml"
sudo virsh define esxi.xml
{% endhighlight %}

### Sources
https://communities.vmware.com/thread/451412
http://mattinaction.blogspot.de/2014/05/install-and-run-full-functional-vmware.html
