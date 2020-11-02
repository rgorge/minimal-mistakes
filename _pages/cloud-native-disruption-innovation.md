---
permalink: /cloud-native-disruption-innovation/
title: Cloud Native as a disruptive innovation for Telco market
comments: true
tags: article cloudnative
---

## _Disclaimer_
_This article is a summary of my personal views and opinions. It does not pretend to be 100% correct as far I'm a human and have my own biases and tend to make mistakes._

## Introduction
The pace of innovation is high and continue to increase. There are very limited amount of industries that were not impacted by Cloud Computing technologies stack. And Telco industry is not one of them. I was witnessing for last 5-7 years how several waves of innovation gone through telecommunications vendors and service providers changing not only products and solutions but organizational structures, processes and ways of working.

The most recent innovation that came to Telco market was Cloud Native technologies. The purpose of this article is to summarize how Cloud Native innovation is different from other multiple changes and improvements happened in industry and explain why it is "disruptive".

What is "disruptive innovation"?
Term "disruptive innovation" is coming from excellent book [The Innovator's Dilemma: When New Technologies Cause Great Firms to Fail by Clayton M. Christensen](https://www.amazon.com/Innovators-Dilemma-Technologies-Management-Innovation/dp/1633691780/ref=pd_lpo_14_t_0/131-5968990-2516511?_encoding=UTF8&pd_rd_i=1633691780&pd_rd_r=7c313654-7e66-4ea5-b137-765688311e31&pd_rd_w=OWxsa&pd_rd_wg=5DFV0&pf_rd_p=7b36d496-f366-4631-94d3-61b87b52511b&pf_rd_r=976ESHN0SD4QY21HEKBZ&psc=1&refRID=976ESHN0SD4QY21HEKBZ).

Before we can jump into "disruptive innovation" discussion, let's first talk about non-disruptive or "sustaining innovation":

> Most new technologies foster improved product performance. I call these sustaining technologies. Some sustaining technologies can be discontinuous or radical in character, while others are of an incremental nature. What all sustaining technologies have In common is that they improve the performance of established products, along the dimensions of performance that mainstream customers in major markets have historically valued.
This type of innovation is very well understood - day-to-day incremental improvements of performance, better utilization of compute resources, new features and accelerated customer experience. This is what investments in R&D and well-define processes bring to the company.

However, innovation can take another form and have completely different on the company and the whole industry.

Below is quote from the The Innovator's Dilemma book that gives high level definition of "disruptive innovation"

>Occasionally, however, disruptive technologies emerge: innovations that result in worse product performance, at least in the near-term.  
Disruptive technologies bring to a market a very different value proposition than had been available previously. Generally, disruptive technologies underperform established products in mainstream markets. But they have other features that a few fringe (and generally new) customers value. Products based on disruptive technologies are typically cheaper, simpler, smaller, and, frequently, more convenient to use.
Why this is important?

Long story short - same patterns and management approaches which lead to product and company success in sustaining technological innovation can cause fail in case of disruptive technological innovation. (Basically, this is what is called Innovators' Dilemma).

And yes, this is a paradox.

Technological innovation in this context means not only new tech as such but also new ways of product development, new value proposition and new product performance characteristics.

For those who are interested in details, I encourage you to read the book. Even some examples in the book are not related to Information Technology at all, the book is extremely valuable for understanding paradox explained above.

## Virtualization and NFV innovation in Telco
Telco industry historically was behind classic IT in enterprises in the area of virtualization technologies. When ETSI started to develop NFV standardizations in late 2012, VMWare was already dominant player in servers virtualization and QEMU/KVM module has been merged into Linux kernel for almost 5 years.

However, it didn't take long for Telco vendors to adopt their application to run on ESXi and KVM hypervisors as Virtual Machines/Virtual Network Functions. For most of the industry players this task didn't require deep changes of architecture or software design. Neither it was necessary to change organizational structure or development process itself. Today we call it "lift and shift" strategy.

NFV ecosystem added new vendors and products into Telco industry - Virtual Infrastructure Managers (VIM, like Red Hat Openstack or VMWare VIO), NFV Infrastructure solutions (like, Juniper Contrail or Cisco VIM), NFV Orchestrators (like, ONAP or OSM) and many others. This added compatibility issues and complicated inter-operability testing but didn't really "disrupt" the industry.

Telco applications didn't become cheaper, simpler or smaller. Value proposition stayed the same to be aligned with Tier 1 service providers expectations and requirements - 99,999% availability, high-performance of individual application instances and etc.

Even the introduction of NFV Orchestration didn't bring a lot of dynamic for applications life-cycle management and resources allocation - it is very difficult to achieve high flexibility for applications that were migrated from hardware appliance to virtualized environment without architecture changes. Monolith stayed monolith. The biggest step that was done in this are was introduction of CUPS (control/user plane separation).

Some architectural and technical decisions were carried over from PNFs (Physical Network Functions) to VNFs, for example:

* Dependency on specific kernel patches and modules which are not part of Linux kernel mainline
* Requirement to have stateful storage or keep state data on the root disk
* Use of proprietary management interface without possibility to fully automate configuration management process
* Proprietary software installation process for VNFs

By all means, virtualization and NFV were sustaining technological innovations. For sure, it brought value for service providers and allowed them to use commodity x86-based servers and build private clouds (mostly, Openstack-based) for Telco applications.

However, most of vendors were ready for this change. Their decision making process and organizational structure didn't require to change but the next wave of innovation was different.

## Cloud Native innovation in Telco
The new technological innovation wave primarily consisted of 3 areas:

* **New virtualization method** - containers instead of VMs
* **New software development architecture principles** - microservices instead of monolith and opensource projects instead of internally developed products
* **New virtual resources orchestration tools** - Kubernetes instead of Openstack

This innovation wave came to Telco and IT almost at the same time, so both industries are moving into the same direction, adopting same tools and best practices. This process erased border between IT and Telco. The most visible change in mentality is 5G Core - Service-Based Architecture, highly distributed design, migration of most part of control-plane protocols to HTTP and REST APIs. All these decisions bring Telco applications closer to "classical" IT products.

New terminology has been created to reflect it - Cloud Native Function. The picture below represents CNCF view on "VNF vs CNF architecture". (in 2019 CNCF established Telecom User Group - [link](https://github.com/cncf/telecom-user-group). There are several interesting programs have been established to drive CNCF projects adoption among Telecom applications vendors and service providers)

![VNF vs CNF architecture](/assets/images/vnf_vs_cnf.png "VNF vs CNF architecture")

Source of the picture - [link](https://docs.google.com/presentation/d/1Bci-rzXnVqlvT2hhSXCBawtsKf8HURC2K4kJSlnB43Q)

From the first view, it looks like just new technology stack, however, there are much more deep changes there.

First of all, containerization of Telco applications cannot be done in the same way as migration from physical appliances to VMs - it is not possible to rely on specific kernel patches, hacks and etc any longer. Telco application must use host OS/worker node kernel simply because there is no kernel in container itself. In addition to that, all Telco CNF processes should be migrated to user-space, including communication with NIC card. In practice it means introduction of DPDK drivers support and integration between Telco application and DPDK. This is one of the reasons why "lift-and-shift" path to containerization not possible for some Telco applications. (However, it is worth to mention Kubevirt and Virtlet technologies that can help with that)

Second, development process is very different between monolith and microservices architectures, so Telco vendor needs to re-architect not only the product but engineering organization as well if they want to build Gold CNF. There can be different new names in new organizational structure - tribes, squads, two-pizza teams and etc., but what they have in common are distributed nature and high autonomous level. This is not an easy task for well-established enterprise with mature internal culture - re-organizations are typically painful and often lead to collateral attrition and damage for people morale.

Of course, it is not mandatory to migrate all Telco applications to microservices, but some level of de-composition is required to meet Cloud Native scalability requirements. Otherwise, why somebody should adopt Cloud without one of it's key advantages - near infinite scale?

The last but not least is expanded ecosystem of Cloud Native products (just look on this landscape! ). Support of Kubernetes is not a great benefit without compatibility with the rest of ecosystem. In order to support this ecosystem (and be able to maintain it), CNFs should be easy to deploy, test and terminate (famous, "pets" vs. "cattle"), but in order to achieve that Telco application should be architected in a new way.

Looks like rocky and challenging path. And, indeed, there were some doubts in industry - will containers provide the same level of the performance? Will networking in Kubernetes provide needed features for Telco CNFs? Why would you re-architect established product when your top 10 customers don't express explicit demand for it? It is a valid question for a company which roadmap is driven by customers' feature requests. And at the same time it is a "disruptive" innovation trap.

Any successful company listen to it's customers and aligns roadmap with customer demands and expectations. The more important customer is (which is equivalent to the amount of money this customer pays), the more influence he has on the roadmap. These two rules combined (plus internal resources constraints) create a situation when company delivers what their top 20 customers want and projects with more ambiguity and risks don't make it into roadmap or always are in N+1 release. Cloud Native technologies early adoption projects fell exactly into such category because it was less predictable, less stable technology without explicit demand from biggest service providers.

Cloud Native technologies allowed new players came into the market and build cheaper, simpler and smaller products. These products maybe underperformed initially and were not ready for Tier 1 service providers, but this has changed rapidly in last couple of years. So, now every company in ICT area is building Cloud Native platform and expect its suppliers to comply with it. For those Telco applications vendors who missed or intentionally post-poned adoption of Cloud Native technologies it is a disruption. Now they need to rush to catch up.

Picture below visualize described above.

![Disruptive and sustaining innovation performance](/assets/images/cloudnative-disruption.png "Disruptive and sustaining innovation performance")

Source of the picture - [link](https://www.google.com/url?sa=i&url=http%3A%2F%2Fweb.mit.edu%2F6.933%2Fwww%2FFall2000%2Fteradyne%2Fclay.html&psig=AOvVaw073DLktrh6ahEnoyHjQClp&ust=1604329250971000&source=images&cd=vfe&ved=0CAMQjB1qFwoTCIid8qrO4ewCFQAAAAAdAAAAABAD)

## Embrace the change
For me there is no doubt that Cloud Native was a "disruptive" innovation for Telecom industry. The process of Cloud Native adoption is moving forward with a high pace. Some Telco vendors will not survive it, others will thrive but the industry will not be the same again.

In the conclusion, couple of resources that I found interesting recently:

1. [Cloud Maturity Matrix](https://info.container-solutions.com/cloud-maturity-matrix) - access organization's maturity in Cloud Native not only from technological point of view but also from culture and processes perspective
2. [Cloud Native Transformation](https://www.amazon.com/Cloud-Native-Transformation-Practical-Innovation/dp/1492048909/ref=sr_1_1?crid=2NLW3R0XDPPKP&dchild=1&keywords=cloud+native+transformation&qid=1604243829&sprefix=cloud+native+t%2Caps%2C230&sr=8-1) - great practical guidelines and best practices for Cloud technologies adoption.


November 2020, Roman Gorge