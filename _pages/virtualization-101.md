---
permalink: /virtualization-101/
title: KVM Virtualization 101
comments: true
tags: article virtualization
---

<blockquote>This is a translation of <a href="https://linkmeup.ru/blog/473.html">my article in Russian</a> that was written in September 2019. 
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

The story of virtualization begins in 1999 when young company VMWare has released product VMWare Workstation. This was a first commercial product that provided virtualization for desktop/client applications. Virtualization of the server part started a little bit later in the form of ESX Server product that evolved in ESXi (i stands for integrated). This product is being used widely in IT and Telco private clouds as hypervisor for server side applications.

<cut/>

On the Opensource side two main projects brought virtualization to Linux:
<ul>
    <li>KVM (Kernel-based Virtual Machine) - linux kernel module that allows kernel to work as hypervisor (i.e. creates infrastructure for launch and management of VMs). It was added in 2.6.20 kernel version back in 2007.</li>
    <li>QEMU (Quick Emulator) - actual emulator of hardware for Virtual Machnine (CPU, Disk, RAM, anything else including USB port). QEMU is used in combination with KVM to achieve near baremetal performance for virtualized applications.</li>
</ul>

<blockquote>
In reality all functionallity available in KVM has been ported to QEMU, but it is not important as far as most part of Linux users do not use QEMU/KVM directly but through at least one layer of abstraction that we will discuss later.
</blockquote>

Сегодня VMWare ESXi и Linux QEMU/KVM это два основных гипервизора, которые доминируют на рынке. Они же являются представителями двух разных типов гипервизоров:
As of today, VMWare ESXi and Linux QEMU/KVM are two most famous hypervisors on the on-premise virtualization market. They are representatives of the same hypervisor type, however, there are two of them:
<ul>
    <li>Type 1 - hypervisor is running on bare-metal hardware. Such hypervisors are VMWare ESXi, Linux KVM, Hyper-V</li>
    <li>Type 2 - hypervisor is running inside Host OS. Such hypervisors are VMWare Workstation are Oracle VirtualBox.</li>
</ul>

Discussion about what hypervisor is better is outside of the scope of this article.

<img src="https://fs.linkmeup.ru/images/adsm/1/1/hypervisors_types.gif" width="600">

Hardware vendors also needed to do their part of the job to provide acceptable level of performance for virtualized applications.

One of the most important and widely used technology is Intel VT (Virtualization Technology) - set of expansions developed by Intel for its x86 CPUs. They are used for efficient hypervisor work (sometimes VT is mandatory. For example, KVM will not work without VT-x and without it hypervisor will need to use only software virtualization without hardware acceleration).

Two most famous of these expansions are VT-x и VT-d. First one is used for CPU performance improvements in virtualization environment because it provides hardware acceleration for some functions (with VT-x 99.9% of Guest OS functions are exectured in physical CPU and emulation is done only when it is required). Second one is needed for attaching physical devices to Virtual Machine (to use SRIOV VF VT-d <a href="https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/virtualization_host_configuration_and_guest_installation_guide/sect-virtualization_host_configuration_and_guest_installation_guide-sr_iov-how_sr_iov_libvirt_works">should be enabled</a> )

Next important concept is difference between full virtualization and para-virtualization.
Full virtualization is good, it allows to run any code on any CPU, however, it is very not efficient and cannot be used for high-load systems.
Para-virtualization, in short, it is when Guest OS is aware that it runs in virtualized environment and cooperates with hypervisor to achieve better performance. I.e. Guest OS - hypervisor interface appears.

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
In theory, QEMU can emulate and CPU type with any flags and functionalities, but in practice either host-model (i.e. the model of physical CPU where hypervisor is running) is used and flags are required flags are enabled before propagating them into VM or named-model (i.e. some specific CPU model, like Intel Cascade Lake, for example) is used.

By default, QEMU will emulate CPU that Guest OS will recognize as QEMU Virtual CPU. It is not the most optimal CPU type for a VM, especially, when application running inside VM relies on specific CPU flags for better performance. <a href="https://wiki.qemu.org/Features/CPUModels" target="_blank">More information about CPU types in QEMU</a>.

QEMU/KVM also allows to control CPU topology, quantity of threads, cache size, vCPU pinning to physical thread and many other things.

What QEMU/KVM features are required depends on application running inside Guest OS. For example, for applications which perform packet processing with high PPS rate it is important to use <b>CPU pinning</b>. This will prevent hypervisor from allocation of physical thread to other VMs and reduce CPU steal time.

<h2>Memory</h2>
The next in line is RAM. From Host OS perspective a VM launched with QEMU/KVM does not differ from any other user-space process (if you run <b>top</b> command in Host OS CLI you will see QEMU processes). This means that memory allocation process for a VM use same Host OS kernel commands as for launching Chrome browser.

<blockquote>
Before we can continue discussion about RAM in VMs, it is required to mention term <b><a href="https://ru.wikipedia.org/wiki/Non-Uniform_Memory_Access" target="_blank">NUMA</a></b> - Non-Uniform Memory Access.
Modern physical servers architecture uses two or more physical CPU (sockets) and associated RAM. This combination of CPU and RAM has a name "NUMA node". Communication between NUMA nodes happens via special bus - <b>QPI</b> (QuickPath Interconnect).
There are local NUMA node - when a process running in operating system uses CPU and RAM inside single NUMA node and remote NUMA node - when a process uses CPU and RAM belonging to different NUMA nodes, i.e. for CPU-RAM communication QPI bus is being used.
</blockquote>

<img src="https://fs.linkmeup.ru/images/adsm/1/1/numa.png" width="600">

From VM point of view, RAM is allocated to it at the moment of Guest OS startup, however, in reality Host OS kernel allocates memory to QEMU/KVM process piece by piece as long as Guest OS requests additional RAM (there are exceptions in this process because it is possible to command QEMU/KVM allocates full memory at the moment of VM creation).

RAM is being allocated not byte by byte but with chunks of specific size - <b>pages</b>. Page size is configurable and in theory can be anything, but in practice it is 4KB (default), 2MB and 1GB. Last two sizes have name <b>HugePages</b> and often being used for memory-intensive virtual machines. A major reason why the one should use Hugepages is <b>Translation Lookaside Buffer</b> (<b><a href="https://en.wikipedia.org/wiki/Translation_lookaside_buffer" target="_blank">TLB</a></b>) - search process that maps page virtual address with physical memory location. TLB process has its own limitations and stores information only about recently used pages. If there is no information about the page in TLB, than <b>TLB miss</b> occures and Host OS CPU should be involved to find physical memory cell that corresponds to the page.

This process is slow and not efficient, that's why less pages with bigger size are used.

QEMU/KVM also allows to emulate various NUMA topologies for Guest OS, allocate memory for a VM only from specific NUMA and etc. The most popular best practice is to allocate memoray for VM from local NUMA node. The reason behind it - avoid additional load on <b>QPI</b> bus that connects CPU sockets of physical server (this is applicable if your server has 2 amd more sockets). 

<hr>

<h1>Storage</h1>
Persistent storage is required for VM to preserve information if VM is rebooted or shutdown.

There are two main types of persistent storage:
<ul>
    <li> Block storage - disk space that can be used for file system installation and partitioning. On the high level you can think about it as simple SSD or HDD disk.</li>
    <li> Object storage - information is stored as file which is available via API. Typical examples of block storage sytems are AWS S3 or Dropbox.</li>
</ul>

Vm needs <b>persistent storage</b>, but how you can achieve it if VM "lives" in memory of Host OS? In short, any call from Guest OS to virtual IDE controller intercepted by QEMU/KVM process and transformed into write operation to physiscal disk attached to Host OS. This method is not efficient from performance perspective, so virtio driver (as in virtual NIC) is used for para-virtualization of IDE or iSCSI devices. This is explained in details <a href="https://www.qemu.org/2018/02/09/understanding-qemu-devices/" target="_blank">here</a>. So, VM call its virtual disk via virtio driver, then QEMU/KVM makes sure that information is written to physical disk. The actual Host OS storage backend can be impelementd as CEPH, NFS or something else. 

The easiest way to emulate persistent storage is to use file in some directory on Host OS as disk space for VM. QEMU/KVM supports multiple formats of such files - raw, vdi, vmdk and etc. However, the most popular one is <b>qcow2</b> (QEMU copy-on-write version 2). In general, qcow2 is a file with defined structure without operating system. A lot of virtual machines are distributed as qcow2 images - VM system disk snapshots packaged in qcow2 format. This approach has number of advantages - qcow2 encoding requires less space than raw snapshot, QEMU/KVM can perform resize operation of qcow2 file, i.e. that it is possible to change VM system disk size, AES encryption is supported as well.

When VM is launched, QEMU/KVM uses qcow2 file as system disk (I intentionally omit VM boot process here) and VM is able to perform read/write operations into qcow2 file via virtio driver. Snapshot creation process works in the same way - qcow2-file represents full copy of VM system disk and by used for backup purposes, evacuation to another hypervisor and etc.

By default, this qcow2 file will be detected in Guest OS as <i>/dev/vda</i> device, Guest OS will perform partitioning and file system installation. Additional qcow2-files attached by QEMU/KVM <i>/dev/vdX</i> devices can be used as <b>block storage</b> in VM (this is exactly how Openstack Cinder works, for example).

<hr>

<h1>Network</h1>
The last but not least there are virutal NIC cards and other I/O devices. Virtual Machine requies a <b>PCI/PCIe-bus</b> for i/O devices connection. QEMU/KVM can emulate different types of chipsets - q35 or i440fx (the first one supports - PCIe, the second - legacy PCI ) and multiple PCI topologies, for example, create separate PCI expander buses for NUMA nodes emulation for a Guest OS.

After PCI/PCIe is created, I/O device should be connected to it. In general, it can be anything from network card to physical GPU. NIC can be fully-virtualized (for example, e1000 interface) or para-virtualized (for, example virtio) or a physical NIC. The last option is applicable for network-intensive applications where line rate PPS rate is required - routers, firewalls and etc.

There are two major approaches for this task - <b>PCI passthrough</b> and <b>SR-IOV</b>. The main difference between them is that with PCI-PT only Guest OS driver is used for NIC, while for SRIOV Host OS driver is required to create and maintain SRIOV Virtual Functions. More details about PCI-PT and SRIOV can be found  <a href="https://www.juniper.net/documentation/en_US/vsrx/topics/concept/security-vsrx-kvm-sr-iov.html" target="_blank">in Juniper article</a>

<img src="https://fs.linkmeup.ru/images/adsm/1/1/sriov.png" width="600">

<blockquote>
It should be mentioned that PCI-PT and SRIOV are complimentary technologies. SRIOV creates Virtual Functions. This is done in Host OS layer and Host OS sees VF as one more PCI/PCIe device.
PCI-PT is a propagation mechanism of PCIe device into the VM. It doesnt matter what Guest OS will do with propagated device.
</blockquote>

So, we discussed main virtual resources types and now it is necessary to understand how VM communicates with outside world via netwrok.
<hr>

<a name="SWITCHING"></a>
<h1>Virtual switching</h1>

Если есть виртуальная машина, а в ней есть виртуальный интерфейс, то, очевидно, возникает задача передачи пакета из одной VM в другую. В Linux-based гипервизорах (KVM, например) эта задача может решаться с помощью Linux bridge, однако, большое распространение получил проект <a href="https://www.openvswitch.org" target="_blank">Open vSwitch</a> (OVS).
Есть несколько основных функциональностей, которые позволили OVS широко распространиться и стать de-facto основным методом коммутации пакетов, который используется во многих платформах облачных вычислений(например, Openstack) и виртуализированных решениях.
<ul>
    <li> Передача сетевого состояния - при миграции VM между гипервизорами возникает задача передачи ACL, QoSs, L2/L3 forwarding-таблиц и прочего. И OVS умеет это.</li>
    <li> Реализация механизма передачи пакетов (datapath) как в kernel, так и в user-space</li>
    <li> CUPS (Control/User-plane separation) архитектура - позволяет перенести функциональность обработки пакетов на специализированный chipset (Broadcom и Marvell chipset, например, могут такое), управляя им через control-plane OVS.</li>
    <li> Поддержка методов удаленного управления трафиком - протокол OpenFlow (привет, SDN).</li>
</ul>

Архитектура OVS на первый взгляд выглядит довольно страшно, но это только на первый взгляд.
<img src="https://fs.linkmeup.ru/images/adsm/1/1/ovs_architecture_01.png" width="600">

Для работы с OVS нужно понимать следующее:
<ul>
<li> <b>Datapath</b> - тут обрабатываются пакеты. Аналогия - switch-fabric железного коммутатора. Datapath включает в себя приём пакетов, обработку заголовков, поиск соответствий по таблице flow, который в Datapath уже запрограммирован. Если OVS работает в kernel, то выполнен в виде модуля ядра. Если OVS работает в user-space, то это процесс в user-space Linux.</li>
<li> <b>vswitchd</b> и <b>ovsdb</b> - демоны в user-space, то что реализует непосредственно сам функциональность коммутатора, хранит конфигурацию, устанавливает flow в datapath и программирует его.</li>
 <li> Набор инструментов для настройки и траблшутинга OVS - <b>ovs-vsctl, ovs-dpctl, ovs-ofctl, ovs-appctl</b>. Все то, что нужно, чтобы прописать в ovsdb конфигурацию портов, прописать какой flow куда должен коммутироваться, собрать статистику и прочее. Добрые люди <a href="https://therandomsecurityguy.com/openvswitch-cheat-sheet/" target="_blank">написали статью</a> по этому поводу.</li>
</ul>

<b>Каким же образом сетевое устройство виртуальной машины оказывается в OVS?</b>

Для решения данной задачи нам необходимо каким-то образом связать между собой виртуальный интерфейс, находящийся в user-space операционной системы с datapath OVS, находящимся в kernel.

В операционной системе Linux передача пакетов между kernel и user-space-процессами осуществляется посредством двух специальных интерфейсов. Оба интерфейса использует запись/чтение пакета в/из специальный файл для передачи пакетов из user-space-процесса в kernel и обратно - file descriptor (FD) (это одна из причин низкой производительности виртуальной коммутации, если datapath OVS находится в kernel - каждый пакет требуется записать/прочесть через FD)

<ul>
    <li><b>TUN</b> (tunnel) - устройство, работающее в L3 режиме и позволяющее записывать/считывать только IP пакеты в/из FD.</li>
    <li> <b>TAP</b> (network tap) - то же самое, что и tun интерфейс + умеет производить операции с Ethernet-фреймами, т.е. работать в режиме L2.</li>
</ul>

<img src="https://fs.linkmeup.ru/images/adsm/1/1/virtual-devices-all.png" width="800">

Именно поэтому при запущенной виртуальной машине в Host OS можно увидеть созданные TAP-интерфейсы командой <i>ip link</i> или <i>ifconfig</i> - это "ответная" часть virtio, которая "видна" в kernel Host OS. Также стоит обратить внимание, что TAP-интерфейс имеет тот же MAC-адрес что и virtio-интерфейс в виртуальной машине.

TAP-интерфейс может быть добавлен в OVS с помощью команд <i>ovs-vsctl</i> - тогда любой пакет, скоммутированный OVS в TAP-интерфейс, будет передан в виртуальную машину через file descriptor.

<blockquote>
    Реальный порядок действий при создании виртуальной машины может быть разным, т.е. сначала можно создать OVS bridge, потом указать виртуальной машине создать интерфейс, соединенный с этим OVS, а можно и наоборот.
</blockquote>

Теперь, если нам необходимо получить возможность передачи пакетов между двумя и более виртуальными машинами, которые запущены на одном гипервизоре, нам потребуется лишь создать OVS bridge и добавить в него TAP-интерфейсы с помощью команд ovs-vsctl. Какие именно команды для этого нужны легко гуглится.

На гипервизоре может быть несколько OVS bridges, например, так работает Openstack Neutron, или же виртуальные машины могут находиться в разных namespace для реализации multi-tenancy.

<b>А если виртуальные машины находятся в разных OVS bridges?</b>

Для решения данной задачи существует другой инструмент - <b>veth pair</b>. Veth pair может быть представлен как пара сетевых интерфейсов, соединенных кабелем - все то, что "влетает" в один интерфейс, "вылетает" из другого. Veth pair используется для соединения между собой нескольких OVS bridges или Linux bridges. Другой важный момент что части veth pair могут находиться в разных namespace Linux OS, то есть veth pair может быть также использован для связи namespace между собой на сетевом уровне.

<a name="INSTRUMENTS"></a>
<h1>Virtualization tools - libvirt, virsh and etc.</h1>

В предыдущих главах мы рассматривали теоретические основы виртуализации, в этой главе мы поговорим об инструментах, которые доступны пользователю непосредственно для запуска и изменения виртуальных машин на KVM-гипервизоре.
Остановимся на трех основных компонентах, которые покрывают 90 процентов всевозможных операций с виртуальными машинами:

<ul>
    <li> libvirt</li>
    <li> virsh CLI</li>
    <li> virt-install</li>
</ul>

<blockquote>
Конечно, существует множество других утилит и CLI-команд, которые позволяют управлять гипервизором, например, можно напрямую пользоваться командами qemu_system_x86_64 или графическим интерфейсом virt manager, но это скорее исключение. К тому же существующие сегодня Cloud-платформы, Openstack, например, используют как раз libvirt.
</blockquote>

<h2>libvirt</h2>
libvirt - это масштабный open-source проект, который занимается разработкой набора инструментов и драйверов для управления гипервизорами. Он поддерживает не только QEMU/KVM, но и ESXi, LXC и много чего еще.
Основная причина его популярности - структурированный и понятный интерфейс взаимодействия через набор XML-файлов плюс возможность автоматизации через API. Стоит оговориться что libvirt не описывает все возможные функции гипервизора, он лишь предоставляет удобный интерфейс использования <b>полезных</b>, с точки зрения участников проекта, функции гипервизора.

И да, libvirt это де-факто стандарт в мире виртуализации сегодня. Только <a href="https://libvirt.org/apps.html" target="_blank"> взгляните на список приложений</a>, которые используют libvirt.

<img src="https://fs.linkmeup.ru/images/adsm/2/libvirt_support.png" width="700">

Хорошая новость про libvirt - все нужные пакеты уже предустановлены во всех наиболее часто используемых Host OS - Ubuntu, CentOS и RHEL, поэтому, скорее всего, собирать руками нужные пакеты и компилировать libvirt вам не придется. В худшем случае придется воспользоваться соответствующим пакетным инсталлятором (apt, yum и им подобные).

При первоначальной установке и запуске libvirt по умолчанию создает Linux bridge virbr0 и его минимальную конфигурацию.

<blockquote>
    Именно поэтому при установке Ubuntu Server, например, вы увидите в выводе команды ifconfig Linux bridge virbr0 - это результат запуска демона libvirtd
</blockquote>

Этот Linux bridge не будет подключен ни к одному физическому интерфейсу, однако, может быть использован для связи виртуальных машин внутри одного гипервизора. Libvirt безусловно может быть использован вместе с OVS, однако, для этого пользователь должен самостоятельно создать OVS bridges с помощью соответствующих OVS-команд.

Любой виртуальный ресурс, необходимый для создания виртуальной машины (compute, network, storage) представлен в виде объекта в libvirt. За процесс описания и создания этих объектов отвечает набор различных XML-файлов.

Детально описывать процесс создания виртуальных сетей и виртуальных хранилищ не имеет особого смысла, так как эта прикладная задача хорошо описана в документации libvirt:

<ul>
    <li> <a href="https://wiki.libvirt.org/page/VirtualNetworking" target="_blank">Networking</a></li>
    <li> <a href="https://libvirt.org/storage.html" target="_blank">Storage</a></li>
</ul>

Сама виртуальная машина со всеми подключенными PCI-устройствами в терминологии libvirt называется domain. Это тоже <a href="https://libvirt.org/formatdomain.html" target="_blank"> объект внутри libvirt</a>, который описывается отдельным XML-файлом.

Этот XML-файл и является, строго говоря, виртуальной машиной со всеми виртуальными ресурсами - оперативная память, процессор, сетевые устройства, диски и прочее. Часто данный XML-файл называют libvirt XML или dump XML.
Вряд ли найдется человек, который понимает все параметры libvirt XML, однако, это и не требуется, когда есть документация.

В общем случае, libvirt XML для Ubuntu Desktop Guest OS будет довольно прост - 40-50 строчек. Поскольку вся оптимизация производительности описывается также в libvirt XML (NUMA-топология, CPU-топологии, CPU pinning и прочее), для сетевых функций libvirt XML может быть очень сложен и содержать несколько сот строк. Любой производитель сетевых устройств, который поставляет свое ПО в виде виртуальных машин, имеет рекомендованные примеры libvirt XML.

<h2>virsh CLI</h2>

Утилита virsh - "родная" командная строка для управления libvirt. Основное ее предназначение - это управление объектами libvirt, описанными в виде XML-файлов. Типичными примерами являются операции start, stop, define, destroy и так далее. То есть жизненный цикл объектов - life-cycle management.

Описание всех команд и флагов virsh также доступно в документации <a href="https://libvirt.org/sources/virshcmdref/html-single/" target="_blank">libvirt</a>.


<h2>virt-install</h2>

Еще одна утилита, которая используется для взаимодействия с libvirt. Одно из основных преимуществ - можно не разбираться с XML-форматом, а обойтись лишь флагами, доступными в virsh-install. Второй важный момент - море примеров и информации в Сети.

Таким образом какой бы утилитой вы ни пользовались, управлять гипервизором в конечном счете будет именно libvirt, поэтому важно понимать архитектуру и принципы его работы.
<hr>

<a name="CONCLUSION"></a>
<h1>Conclusion</h1>

В данной статье мы рассмотрели минимальный набор теоретических знаний, который необходим для работы с виртуальными машинами. Я намеренно не приводил практических примеров и выводов команд, поскольку таких примеров можно найти сколько угодно в Сети, и я не ставил перед собой задачу написать "step-by-step guide". Если вас заинтересовала какая-то конкретная тема или технология, оставляйте свои комментарии и пишите вопросы.
<hr>

<a name="LINKS"></a>
<h1>Useful links</h1>
<ul>
    <li><a href="https://www.qemu.org/2018/02/09/understanding-qemu-devices/" target="_blank">Understanding QEMU Devices</a>.</li>
    <li><a href="https://www.juniper.net/documentation/en_US/vsrx/topics/concept/security-vsrx-kvm-sr-iov.html" target="_blank">KVM/SR-IOV</a>.</li>
</ul>
<hr>