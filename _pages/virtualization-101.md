---
permalink: /virtualization-101/
title: Cloud Native as a disruptive innovation for Telco market
comments: true
tags: article virtualization
---

<blockquote>This is a translation of <a href="https://linkmeup.ru/blog/473.html">original article in Russian</a> that was written in September 2019. 
Translated and published with blessing from <a href="https://www.linkedin.com/in/marat-sibgatulin/">Marat Sibgatulin</a>
</blockquote>
<hr>

Virtualization is so deep and broad subject, that it is not possible to cover all details of hypervisor (and not needed, actually). I will concentrate on "minimal valuable pack" of knowledge that is required to understand any virtualized solution, not necessarily Telco.

<a href="https://fs.linkmeup.ru/images/adsm/1/1/kdpv.png" target="_blank"><img src="https://fs.linkmeup.ru/images/adsm/1/1/kdpv.png" width="800"></a>

<h1>Содержание</h1>
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
Из чего же состоит виртуальная машина?
Выделяют три основных вида виртуальных ресурсов:

<ul>
    <li> compute - процессор и оперативная память</li>
    <li> storage - системный диск виртуальной машины и блочные хранилища</li>
    <li> network - сетевые карты и устройства ввода/вывода</li>
</ul>

<hr>

<h1>Compute</h1>


<h2>CPU</h2>
Теоретически QEMU способен эмулировать любой тип процессора и соотвествующие ему флаги и функциональность, на практике используют либо host-model и точечно выключают флаги перед передачей в Guest OS либо берут named-model и точечно включают\выключают флаги.

По умолчанию QEMU будет эмулировать процессор, который будет распознан Guest OS как QEMU Virtual CPU. Это не самый оптимальный тип процессора, особенно если приложение, работающее в виртуальной машине, использует CPU-флаги для своей работы. <a href="https://wiki.qemu.org/Features/CPUModels" target="_blank">Подробнее о разных моделях CPU в QEMU</a>.

QEMU/KVM также позволяет контролировать топологию процессора, количество тредов, размер кэша, привязывать vCPU к физическому ядру и много чего еще.

Нужно ли это для виртуальной машины или нет, зависит от типа приложения, работающего в Guest OS. Например, известный факт, что для приложений, выполняющих обработку пакетов с высоким PPS, важно делать <b>CPU pinning</b>, то есть не позволять передавать физический процессор другим виртуальным машинам.

<h2>Memory</h2>
Далее на очереди оперативная память - RAM. С точки зрения Host OS запущенная с помощью QEMU/KVM виртуальная машина ничем не отличается от любого другого процесса, работающего в user-space операционной системы. Соотвественно и процесс выделения памяти виртуальной машине выполняется теми же вызовами в kernel Host OS, как если бы вы запустили, например, Chrome браузер.

<blockquote>
Перед тем как продолжить повествование об оперативной памяти в виртуальных машинах, необходимо сделать отступление и объяснить термин <b><a href="https://ru.wikipedia.org/wiki/Non-Uniform_Memory_Access" target="_blank">NUMA</a></b> - Non-Uniform Memory Access.
Архитектура современных физических серверов предполагает наличие двух или более процессоров (CPU) и ассоциированной с ней оперативной памятью (RAM). Такая связка процессор + память называется узел или нода (node). Связь между различными NUMA nodes осуществляется посредством специальной шины - <b>QPI</b> (QuickPath Interconnect)

Выделяют локальную NUMA node - когда процесс, запущенный в операционной системе, использует процессор и оперативную память, находящуюся в одной NUMA node, и удаленную NUMA node - когда процесс, запущенный в операционной системе, использует процессор и оперативную память, находящиеся в разных NUMA nodes, то есть для взаимодействия процессора и памяти требуется передача данных через QPI шину.
</blockquote>

<img src="https://fs.linkmeup.ru/images/adsm/1/1/numa.png" width="600">

С точки зрения виртуальной машины память ей уже выделена на момент ее запуска, однако в реальности это не так, и kernel Host OS выделяет процессу QEMU/KVM новые участки памяти по мере того как приложение в Guest OS запрашивает дополнительную память (хотя тут тоже может быть исключение, если прямо указать QEMU/KVM выделить всю память виртуальной машине непосредственно при запуске).

Память выделяется не байт за байтом, а определенным размером - <b>page</b>. Размер page конфигурируем и теоретически может быть любым, но на практике используется размер 4kB (по умолчанию), 2MB и 1GB. Два последних размера называются <b>HugePages</b> и часто используются для выделения памяти для memory intensive виртуальных машин. Причина использования HugePages в процессе поиска соответствия между виртуальным адресом page и физической памятью в <b>Translation Lookaside Buffer</b> (<b><a href="https://en.wikipedia.org/wiki/Translation_lookaside_buffer" target="_blank">TLB</a></b>), который в свою очередь ограничен и хранит информацию только о последних использованных pages. Если информации о нужной page в TLB нет, происходит процесс, называемый <b>TLB miss</b>, и требуется задействовать процессор Host OS для поиска ячейки физической памяти, соответствующей нужной page.

Данный процесс неэффективен и медлителен, поэтому и используется меньшее количество pages бо́льшего размера.
QEMU/KVM также позволяет эмулировать различные NUMA-топологии для Guest OS, брать память для виртуальной машины только из определенной NUMA node Host OS и так далее. Наиболее распространенная практика - брать память для виртуальной машины из NUMA node локальной по отношению к процессорам, выделенным для виртуальной машины. Причина - желание избежать лишней нагрузки на <b>QPI</b> шину, соединяющую CPU sockets физического сервера (само собой, это логично если в вашем сервере 2 и более sockets).

<hr>

<h1>Storage</h1>
Как известно, оперативная память потому и называется оперативной, что ее содержимое исчезает при отключении питания или перезагрузке операционной системы. Чтобы хранить информацию, требуется постоянное запоминающее устройство (ПЗУ) или <b>persistent storage</b>.
Существует два основных вида persistent storage:
<ul>
    <li> Block storage (блоковое хранилище) - блок дискового пространства, который может быть использован для установки файловой системы и создания партиций. Если грубо, то можно воспринимать это как обычный диск.</li>
    <li> Object storage (объектное хранилище) - информация может быть сохранена только в виде объекта (файла), доступного по HTTP/HTTPS. Типичными примерами объектного хранилища являются AWS S3 или Dropbox.</li>
</ul>

Виртуальная машина нуждается в <b>persistent storage</b>, однако, как это сделать, если виртуальная машина "живет" в оперативной памяти Host OS? Если вкратце, то любое обращение Guest OS к контроллеру виртуального диска перехватывается QEMU/KVM и трансформируется в запись на физический диск Host OS. Этот метод неэффективен, и поэтому здесь так же как и для сетевых устройств используется virtio-драйвер вместо полной эмуляции IDE или iSCSI-устройства. Подробнее об этом можно почитать <a href="https://www.qemu.org/2018/02/09/understanding-qemu-devices/" target="_blank">здесь</a>. Таким образом виртуальная машина обращается к своему виртуальному диску через virtio-драйвер, а далее QEMU/KVM делает так, чтобы переданная информация записалась на физический диск. Важно понимать, что в Host OS дисковый backend может быть реализован в виде CEPH, NFS или iSCSI-полки.

Наиболее простым способом эмулировать persistent storage является использование файла в какой-либо директории Host OS как дискового пространства виртуальной машины. QEMU/KVM поддерживает множество различных форматов такого рода файлов - raw, vdi, vmdk и прочие. Однако наибольшее распространение получил формат <b>qcow2</b> (QEMU copy-on-write version 2). В общем случае, qcow2 представляет собой определенным образом структурированный файл без какой-либо операционной системы. Большое количество виртуальных машин распространяется именно в виде qcow2-образов (images) и являются копией системного диска виртуальной машины, упакованной в qcow2-формат. Это имеет ряд преимуществ - qcow2-кодирование занимает гораздо меньше места, чем raw копия диска байт в байт, QEMU/KVM умеет изменять размер qcow2-файла (resizing), а значит имеется возможность изменить размер системного диска виртуальной машины, также поддерживается AES шифрование qcow2 (это имеет смысл, так как образ виртуальной машины может содержать интеллектуальную собственность).

Далее, когда происходит запуск виртуальной машины, QEMU/KVM использует qcow2-файл как системный диск (процесс загрузки виртуальной машины я опускаю здесь, хотя это тоже является интересной задачей), а виртуальная машина имеет возможность считать/записать данные в qcow2-файл через virtio-драйвер. Таким образом и работает процесс снятия образов виртуальных машин, поскольку в любой момент времени qcow2-файл содержит полную копию системного диска виртуальной машины, и образ может быть использован для резервного копирования, переноса на другой хост и прочее.

В общем случае этот qcow2-файл будет определяться в Guest OS как <i>/dev/vda</i>-устройство, и Guest OS произведет разбиение дискового пространства на партиции и установку файловой системы. Аналогично, следующие qcow2-файлы, подключенные QEMU/KVM как <i>/dev/vdX</i> устройства, могут быть использованы как <b>block storage</b> в виртуальной машине для хранения информации (именно так и работает компонент Openstack Cinder).
<hr>

<h1>Network</h1>
Последним в нашем списке виртуальных ресурсов идут сетевые карты и устройства ввода/вывода. Виртуальная машина, как и физический хост, нуждается в <b>PCI/PCIe-шине</b> для подключения устройств ввода/вывода. QEMU/KVM способен эмулировать разные типы чипсетов - q35 или i440fx (первый поддерживает - PCIe, второй - legacy PCI ), а также различные PCI-топологии, например, создавать отдельные PCI-шины (PCI expander bus) для NUMA nodes Guest OS.

После создания PCI/PCIe шины необходимо подключить к ней устройство ввода/вывода. В общем случае это может быть что угодно - от сетевой карты до физического GPU. И, конечно же, сетевая карта, как полностью виртуализированная (полностью виртуализированный интерфейс e1000, например), так и пара-виртуализированная (virtio, например) или физическая NIC. Последняя опция используется для data-plane виртуальных машин, где требуется получить line-rate скорости передачи пакетов - маршрутизаторов, файрволов и тд.

Здесь существует два основных подхода - <b>PCI passthrough</b> и <b>SR-IOV</b>. Основное отличие между ними - для PCI-PT используется драйвер только внутри Guest OS, а для SRIOV  используется драйвер Host OS (для создания <b>VF - Virtual Functions</b>) и драйвер Guest OS для управления SR-IOV VF. Более подробно об PCI-PT и SRIOV отлично <a href="https://www.juniper.net/documentation/en_US/vsrx/topics/concept/security-vsrx-kvm-sr-iov.html" target="_blank">написал Juniper</a>.

<img src="https://fs.linkmeup.ru/images/adsm/1/1/sriov.png" width="600">

<blockquote>
Для уточнения стоит отметить что, PCI passthrough  и SR-IOV  это дополняющие друг друга технологии. SR-IOV это нарезка физической функции на виртуальные функции. Это выполняется на уровне Host OS. При этом Host OS видит виртуальные функции как еще одно PCI/PCIe устройство. Что он дальше с ними делает - не важно.

А PCI-PT это механизм проброса любого Host OS PCI устройства в Guest OS, в том числе виртуальной функции, созданной SR-IOV устройством
</blockquote>

Таким образом мы рассмотрели основные виды виртуальных ресурсов и следующим шагом необходимо понять как виртуальная машина общается с внешним миром через сеть.
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