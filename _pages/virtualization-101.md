---
permalink: /virtualization-101/
title: KVM Virtualization 101
comments: true
tags: article virtualization
---

<blockquote>This is a translation of <a href="https://linkmeup.ru/blog/473.html">my article in Russian</a> that was written by me in September 2019. 
Translated and published with blessing from <a href="https://www.linkedin.com/in/marat-sibgatulin/">Marat Sibgatulin</a>
</blockquote>
<hr>

Virtualization is so deep and broad subject, that it is not possible to cover all details of hypervisor (and not needed, actually). I will concentrate on "minimal valuable pack" of knowledge that is required to understand any virtualized solution, not necessarily Telco.

<a href="https://fs.linkmeup.ru/images/adsm/1/1/kdpv.png" target="_blank"><img src="https://fs.linkmeup.ru/images/adsm/1/1/kdpv.png" width="800"></a>

<h1>Content</h1>
<ul>
    <li><b><a href="#INTRODUCTION">Overview and brief history of virtualization technology</a></b></li>
    <li><b><a href="#RESOURCES">Types of virtualized resources - compute, storage, network</a></b></li>
    <li><b><a href="#SWITCHING">Virtual switching</a></b></li>
    <li><b><a href="#INSTRUMENTS">Virtualization tools - libvirt, virsh and etc.</a></b></li>
    <li><b><a href="#CONCLUSION">Conclusion</a></b></li>
</ul>
<hr>


<a name="INTRODUCTION"></a>
<h1>Overview and brief history of virtualization technology</h1>

The story of virtualization begins in 1999 when young company VMWare has released product VMWare Workstation. This was a first commercial product that provided virtualization for desktop/client applications. Virtualization of the server part started a little bit later in the form of ESX Server product that evolved in ESXi (i stands for integrated). This product is being used widely in IT and Telco private clouds as hypervisor for server-side applications.

<cut/>

On the Opensource side two main projects brought virtualization to Linux:
<ul>
    <li>KVM (Kernel-based Virtual Machine) - linux kernel module that allows kernel to work as hypervisor (i.e., creates infrastructure for launch and management of VMs). It was added in 2.6.20 kernel version back in 2007.</li>
    <li>QEMU (Quick Emulator) - actual emulator of hardware for Virtual Machnine (CPU, Disk, RAM, anything else including USB port). QEMU is used in combination with KVM to achieve near baremetal performance for virtualized applications.</li>
</ul>

<blockquote>
In reality all functionality available in KVM has been ported to QEMU, but it is not important as far as most part of Linux users do not use QEMU/KVM directly but through at least one layer of abstraction that we will discuss later.
</blockquote>

As of today, VMWare ESXi and Linux QEMU/KVM are two most famous hypervisors on the on-premise virtualization market. They are representatives of the same hypervisor type, however, there are two of them:
<ul>
    <li>Type 1 - hypervisor is running on bare-metal hardware. Such hypervisors are VMWare ESXi, Linux KVM, Hyper-V</li>
    <li>Type 2 - hypervisor is running inside Host OS. Such hypervisors are VMWare Workstation are Oracle VirtualBox.</li>
</ul>

Discussion about what hypervisor is better is outside of the scope of this article.

<img src="https://fs.linkmeup.ru/images/adsm/1/1/hypervisors_types.gif" width="600">

Hardware vendors also needed to do their part of the job to provide acceptable level of performance for virtualized applications.

One of the most important and widely used technology is Intel VT (Virtualization Technology) - set of expansions developed by Intel for its x86 CPUs. They are used for efficient hypervisor work (sometimes VT is mandatory. For example, KVM will not work without VT-x and without it hypervisor will need to use only software virtualization without hardware acceleration).

Two most famous of these expansions are VT-x и VT-d. First one is used for CPU performance improvements in virtualization environment because it provides hardware acceleration for some functions (with VT-x 99.9% of Guest OS functions are executed in physical CPU and emulation is done only when it is required). Second one is needed for attaching physical devices to Virtual Machine (to use SRIOV VF VT-d <a href="https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/virtualization_host_configuration_and_guest_installation_guide/sect-virtualization_host_configuration_and_guest_installation_guide-sr_iov-how_sr_iov_libvirt_works">should be enabled</a> )

Next important concept is difference between full-virtualization and para-virtualization.
Full virtualization is good, it allows to run any code on any CPU, however, it is very not efficient and cannot be used for high-load systems.
Para-virtualization, in short, it is when Guest OS is aware that it runs in virtualized environment and cooperates with hypervisor to achieve better performance. I.e., Guest OS - hypervisor interface appears.

Overwhelming majority of operating systems support para-virtualization - in Linux kernel it is supported from version 2.6.20.

Virtual Machnine needs not only vCPU and RAM but PCI devices as well. It needs drivers to operate virtual network cards, disks and etc.
Linux KVM solved this with introduction of <b>virtio</b> - framework for development and operation of virtualized of I/O devices.
Virtio is an additional layer of abstraction that allows emulate I/O device in para-virtualized mode. It provides unified interface towards VM. This allows use virtio for different emulated devices.
Virtio consists of:

<ul>
    <li> Front-end driver - part that resides in VM</li>
    <li> Back-end driver - part that resides in hypervisor</li>
    <li> Transport driver - communication layer between front-end and back-end</li>
</ul>
Modular approach allows to change technologies in hypervisor without touching drivers of virtual machine. This is important for various network acceleration technologies like OVS-DPDK or VPP.
<blockquote>
If you ever answered a question in RFP "Does your application support virtio?" it was about front-end part of virtio drivers.
</blockquote>

<a name="RESOURCES"></a>
<h1>Types of virtualized resources - compute, storage, network</h1>
So, what components a VM consist of?
Thre are three most important virtual resources:

<ul>
    <li> compute - CPU and RAM</li>
    <li> storage - VM root disk and attached block storage devices</li>
    <li> network - virtual network cards and other I/O devices</li>
</ul>

<hr>

<h1>Compute</h1>


<h2>CPU</h2>
In theory, QEMU can emulate and CPU type with any flags and functionalities, but in practice either host-model (i.e., the model of physical CPU where hypervisor is running) is used and flags are required flags are enabled before propagating them into VM or named-model (i.e., some specific CPU model, like Intel Cascade Lake, for example) is used.

By default, QEMU will emulate CPU that Guest OS will recognize as QEMU Virtual CPU. It is not the most optimal CPU type for a VM, especially, when application running inside VM relies on specific CPU flags for better performance. <a href="https://wiki.qemu.org/Features/CPUModels" target="_blank">More information about CPU types in QEMU</a>.

QEMU/KVM also allows to control CPU topology, quantity of threads, cache size, vCPU pinning to physical thread and many other things.

What QEMU/KVM features are required depends on application running inside Guest OS. For example, for applications which perform packet processing with high PPS rate it is important to use <b>CPU pinning</b>. This will prevent hypervisor from allocation of physical thread to other VMs and reduce CPU steal time.

<h2>Memory</h2>
The next in line is RAM. From Host OS perspective a VM launched with QEMU/KVM does not differ from any other user-space process (if you run <b>top</b> command in Host OS CLI you will see QEMU processes). This means that memory allocation process for a VM use same Host OS kernel commands as for launching Chrome browser.

<blockquote>
Before we can continue discussion about RAM in VMs, it is required to mention term <b><a href="https://ru.wikipedia.org/wiki/Non-Uniform_Memory_Access" target="_blank">NUMA</a></b> - Non-Uniform Memory Access.
Modern physical servers' architecture uses two or more physical CPU (sockets) and associated RAM. This combination of CPU and RAM has a name "NUMA node". Communication between NUMA nodes happens via special bus - <b>QPI</b> (QuickPath Interconnect).
There is local NUMA node - when a process running in operating system uses CPU and RAM inside single NUMA node and remote NUMA node - when a process uses CPU and RAM belonging to different NUMA nodes, i.e. for CPU-RAM communication QPI bus is being used.
</blockquote>

<img src="https://fs.linkmeup.ru/images/adsm/1/1/numa.png" width="600">

From VM point of view, RAM is allocated to it at the moment of Guest OS startup, however, in reality Host OS kernel allocates memory to QEMU/KVM process piece by piece as long as Guest OS requests additional RAM (there are exceptions in this process because it is possible to command QEMU/KVM allocates full memory at the moment of VM creation).

RAM is being allocated not byte by byte but with chunks of specific size - <b>pages</b>. Page size is configurable and in theory can be anything, but in practice it is 4KB (default), 2MB and 1GB. Last two sizes have name <b>HugePages</b> and often being used for memory-intensive virtual machines. A major reason why the one should use Hugepages is <b>Translation Lookaside Buffer</b> (<b><a href="https://en.wikipedia.org/wiki/Translation_lookaside_buffer" target="_blank">TLB</a></b>) - search process that maps page virtual address with physical memory location. TLB process has its own limitations and stores information only about recently used pages. If there is no information about the page in TLB, then <b>TLB miss</b> occurs and Host OS CPU should be involved to find physical memory cell that corresponds to the page.

This process is slow and not efficient, that's why less pages with bigger size are used.

QEMU/KVM also allows to emulate various NUMA topologies for Guest OS, allocate memory for a VM only from specific NUMA and etc. The most popular best practice is to allocate memory for VM from local NUMA node. The reason behind it - avoid additional load on <b>QPI</b> bus that connects CPU sockets of physical server (this is applicable if your server has 2 amd more sockets). 

<hr>

<h1>Storage</h1>
Persistent storage is required for VM to preserve information if VM is rebooted or shutdown.

There are two main types of persistent storage:
<ul>
    <li> Block storage - disk space that can be used for file system installation and partitioning. On the high level you can think about it as simple SSD or HDD disk.</li>
    <li> Object storage - information is stored as file which is available via API. Typical examples of block storage systems are AWS S3 or Dropbox.</li>
</ul>

VM needs <b>persistent storage</b>, but how you can achieve it if VM "lives" in memory of Host OS? In short, any call from Guest OS to virtual IDE controller intercepted by QEMU/KVM process and transformed into write operation to physiscal disk attached to Host OS. This method is not efficient from performance perspective, so virtio driver (as in virtual NIC) is used for para-virtualization of IDE or iSCSI devices. This is explained in details <a href="https://www.qemu.org/2018/02/09/understanding-qemu-devices/" target="_blank">here</a>. So, VM call its virtual disk via virtio driver, then QEMU/KVM makes sure that information is written to physical disk. The actual Host OS storage backend can be impelemented as CEPH, NFS or something else. 

The easiest way to emulate persistent storage is to use file in some directory on Host OS as disk space for VM. QEMU/KVM supports multiple formats of such files - raw, vdi, vmdk and etc. However, the most popular one is <b>qcow2</b> (QEMU copy-on-write version 2). In general, qcow2 is a file with defined structure without operating system. A lot of virtual machines are distributed as qcow2 images - VM system disk snapshots packaged in qcow2 format. This approach has number of advantages - qcow2 encoding requires less space than raw snapshot, QEMU/KVM can perform resize operation of qcow2 file, i.e., that it is possible to change VM system disk size, AES encryption is supported as well.

When VM is launched, QEMU/KVM uses qcow2 file as system disk (I intentionally omit VM boot process here) and VM is able to perform read/write operations into qcow2 file via virtio driver. Snapshot creation process works in the same way - qcow2-file represents full copy of VM system disk and by used for backup purposes, evacuation to another hypervisor and etc.

By default, this qcow2 file will be detected in Guest OS as <i>/dev/vda</i> device, Guest OS will perform partitioning and file system installation. Additional qcow2-files attached by QEMU/KVM <i>/dev/vdX</i> devices can be used as <b>block storage</b> in VM (this is exactly how Openstack Cinder works, for example).

<hr>

<h1>Network</h1>
The last but not least there are virtual NIC cards and other I/O devices. Virtual Machine requies a <b>PCI/PCIe-bus</b> for i/O devices connection. QEMU/KVM can emulate different types of chipsets - q35 or i440fx (the first one supports - PCIe, the second - legacy PCI) and multiple PCI topologies, for example, create separate PCI expander buses for NUMA nodes emulation for a Guest OS.

After PCI/PCIe is created, I/O device should be connected to it. In general, it can be anything from network card to physical GPU. NIC can be fully-virtualized (for example, e1000 interface) or para-virtualized (for, example virtio) or a physical NIC. The last option is applicable for network-intensive applications where line rate PPS rate is required - routers, firewalls and etc.

There are two major approaches for this task - <b>PCI passthrough</b> and <b>SR-IOV</b>. The main difference between them is that with PCI-PT only Guest OS driver is used for NIC, while for SRIOV Host OS driver is required to create and maintain SRIOV Virtual Functions. More details about PCI-PT and SRIOV can be found  <a href="https://www.juniper.net/documentation/en_US/vsrx/topics/concept/security-vsrx-kvm-sr-iov.html" target="_blank">in Juniper article</a>

<img src="https://fs.linkmeup.ru/images/adsm/1/1/sriov.png" width="600">

<blockquote>
It should be mentioned that PCI-PT and SRIOV are complementary technologies. SRIOV creates Virtual Functions. This is done in Host OS layer and Host OS sees VF as one more PCI/PCIe device.
PCI-PT is a propagation mechanism of PCIe device into the VM. It doesnt matter what Guest OS will do with propagated device.
</blockquote>

So, we discussed main virtual resources types and now it is necessary to understand how VM communicates with outside world via netwrok.
<hr>

<a name="SWITCHING"></a>
<h1>Virtual switching</h1>

We have now Virtual Machine with virtual NIC, so now we have a task to send packets from VM to another or from VM to outside world. In KVM hypervisor this task can be done with Linux bridge, however, we are going to discuss another famous technology - <a href="https://www.openvswitch.org" target="_blank">Open vSwitch</a> (OVS).

There are several features that made OVS become de-facto standard technology for virtual switching in hypervisors. OVS is widely used in private cloud - Openstack, for example.

<ul>
    <li> Network state transition - OVS supports transition of ACL, QoS and L2/L3 forwarding tables between hypervisors when VM is migrated</li>
    <li> Datapath implementation in kernel and user-space</li>
    <li> CUPS (Control/User-plane separation) architecture - move packet processing to dedicated hardware (for example, Broadcom or Marvell chipsets) and manage it with OVS control-plane.</li>
    <li> OpenFlow protocol support for remote flow control (hello, SDN).</li>
</ul>
OVS architecture can be confusing, but it is very logical and straightforward.
<img src="https://fs.linkmeup.ru/images/adsm/1/1/ovs_architecture_01.png" width="600">

There are key OVS principles that should be understood before going deeper:
<ul>
<li> <b>Datapath</b> - packet processing part. Good analogy is a switch-fabric of physical switch. Datapath does headers processing of incoming packets and flow table search process. If OVS works in kernel mode, datapath works in kernel-space. If OVS works as user-space, datapath works as user-space process as well. </li>
<li> <b>vswitchd</b> и <b>ovsdb</b> - key daemons in OVS. They implement switching function itself, store configuration and provision flow information into datapath. </li>
 <li> Key instruments for OVS configuration and troubleshooting - <b>ovs-vsctl, ovs-dpctl, ovs-ofctl, ovs-appctl</b>. These tools are required to write ports configuration into ovsdb, add flow information, collect statistics and etc. There is a very <a href="https://therandomsecurityguy.com/openvswitch-cheat-sheet/" target="_blank">good article</a> with more details about OVS tools.</li>
</ul>

<b>So, how exactly virtual NIC of VM appears in OVS?</b>

In order to solve this task, it is required to connect virtual interface (which lives in user-space) emulated by QEMU/KVM with OVS datapath (which lives in kernel-space).

Linux kernel supports packets exchange between kernel and user-space processes via two special interfaces. Both interfaces read/write packets to/from special file descriptor (FD) to create network communication between kernel and user-space process. (this is one of the reasons why virtual switching is slow when OVS runs in kernel mode - every packet should be read/written via FD).

<ul>
    <li><b>TUN</b> (tunnel) - interface that works in L3 mode. It can read/write only IP packets via FD.</li>
    <li> <b>TAP</b> (network tap) - same as tun interface + it can work with Ethernet frames, i.e., work in L2 mode.</li>
</ul>

<img src="https://fs.linkmeup.ru/images/adsm/1/1/virtual-devices-all.png" width="800">

This is the reason why in Host OS CLI commands <i>ip link</i> или <i>ifconfig</i> output you can see TAP interfaces - this is the second part of virtio that "lives" in Host OS kernel. It is also important to mention that TAP interface has the same MAC-address as vitio interface inside Virtual Machine.

TAP interface can be added into OVS with command <i>ovs-vsctl</i>. In this case any packet that switched by OVS to TAP interface will be send to VM via file descriptor write/read operation.

<blockquote>
    The actual order of VM creation can vary, e.g. that first OVS and then VM interface will be created and attached to OVS or it can be be vice versa.
</blockquote>

Now it is we need to implement network connectivity between two or more VMs running on the same hypervisor. In order to do this, we need to create single instance of OVS bridge and add TAP interfaces into it with ovs-vsctl command. Exact syntaxis of the command is very easy to google.

Hypervisor can have several OVS bridges for different purposes (this is for example is used for Openstack Neutron) or VMs can use multiple namespaces for multi-tenancy.

<b>But what to do if VMs are connected to different OVS bridges?</b>

Another instrument is used for this task - <b>veth pair</b>. Veth pair can be visualized as a pair of network interfaces connected with cable. Everything that enters one interface, will exit from another side. Veth pair is used to connect several instances of OVS or Linux bridges. Another important feature is that different parts of veth pair can exist in different Linux namespaces, e.g. veth pair can connect namespaces on network layer.

<a name="INSTRUMENTS"></a>
<h1>Virtualization tools - libvirt, virsh and etc.</h1>

In previous chapter we discussed virtualization theory. In this chapter we will talk about key tools that can be used for virtual machine management on KVM hypervisor.
There are 3 major tools that cover 90% of VMs lifecycle and troubleshooting operations: 

<ul>
    <li> libvirt</li>
    <li> virsh CLI</li>
    <li> virt-install</li>
</ul>

<blockquote>
Of course, there are other tools and CLI commands that allows to manage hypervisor. For example, you can use qemu_system_x86_64 CLI commands or virt manager user interface, but it is rare case. Many Private Cloud platforms, like Openstack, utilize libvirt.
</blockquote>

<h2>libvirt</h2>
libvirt is a big open-source project that develops tools and drivers for hypervisor management. It supports not only QEMU/KVM but ESXi, LXC and other virtualization platforms.
One of the key libvirt benefits - well-structured and human-readable format of XML files that define Virtual Machine parameters. It should be mentioned that libvirt does not describe all possible hypervisor features, but provides easy to use interface to <b>most important</b> features.

So, libvirt is de-facto virtualization standard. Look on the <a href="https://libvirt.org/apps.html" target="_blank"> list</a> of supported applications!

<img src="https://fs.linkmeup.ru/images/adsm/2/libvirt_support.png" width="700">

libvirt by default creates and configures Linux bridge virbr0 during installation process.

<blockquote>
    This is the reason why after finishing installation of Ubuntu Server, for example, you will see Linux bridge virbr0 in the output of ip link command - it is created by libvirtd daemon.
</blockquote>

This default Linux bridge will not be connected to any physical interface; however, it can be used for connectivity between VMs running on the same hypervisor. Libvirt, for sure, can be used together with OVS but OVS bridges should be created first with corresponding CLI commands.

Any virtual resource required for VM creation (compute, network, storage) is a libvirt object. This object described in number of XML files.

XML files structure and syntaxis explained very good in libvirt documentation, so we will omit it in this article.

<ul>
    <li> <a href="https://wiki.libvirt.org/page/VirtualNetworking" target="_blank">Networking</a></li>
    <li> <a href="https://libvirt.org/storage.html" target="_blank">Storage</a></li>
</ul>

Virtual Machine with connected PCI devices called domain in libvirt terminology. Domain is also <a href="https://libvirt.org/formatdomain.html" target="_blank"> libvirt object</a> and it is described as separate XML file.

This domain XML file is a VM with all virtual resource - memory, CPU, NICs, storage devices and etc. This XML file called libvirt XML or dump XML. I don't think that there is a person who knows all libvirt XML parameters but thanks to documentation it is unnecessary

In general case, libvirt XML for Ubuntu Desktop Guest OS will be short - 40-50 lines. However, when it comes to creation of highly-optimized VMs with NUMA topology, CPU topology, CPU pinning and other features, libvirt XML can grow to several hundreds rows. Any vendor that supplies products as VMs have a recommended example of libvirt XMLs.

<h2>virsh CLI</h2>

virsh tool is a native libvirt CLI. Its main purpose is to manage libvirt objects described by XML files. Typical examples are start, stop, define, destroy and other lifecycle management operations.

All commands and flags of virsh CLI explained in <a href="https://libvirt.org/sources/virshcmdref/html-single/" target="_blank">libvirt</a> documentation.

<h2>virt-install</h2>

One more tool that is used together with libvirt. One of the key benefits - no need to deal with XML format and use only CLI flags available in virt-install plus huge amount of examples in Internet.

It doesnt matter what tool you will use, the actual hypervisor management will be done by libvirt.
<hr>

<a name="CONCLUSION"></a>
<h1>Conclusion</h1>

This article covers minimum theoritical knowledge needed to work with Virtual Machines. I intentionally did not mention practical examples or commands outputs because there are huge amount of examples and step-by-step guides available already.
<hr>

<a name="LINKS"></a>
<h1>Useful links</h1>
<ul>
    <li><a href="https://www.qemu.org/2018/02/09/understanding-qemu-devices/" target="_blank">Understanding QEMU Devices</a>.</li>
    <li><a href="https://www.juniper.net/documentation/en_US/vsrx/topics/concept/security-vsrx-kvm-sr-iov.html" target="_blank">KVM/SR-IOV</a>.</li>
</ul>
<hr>