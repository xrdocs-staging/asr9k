---
published: true
date: '2024-08-05 14:53 +0200'
title: Reaching Full BNG Scale on ASR 9000
tags:
  - ASR 9000
  - BNG
  - QoS
position: hidden
excerpt: This article covers ASR 9000 best practices to maximize BNG scale
author: Cisco Design Team
---
# Introduction

Did you ever wonder how does ASR 9000 platform manage Broadband Network Gateway (BNG) function with Quality of Service (QoS) applied? Did you ever want to increase subscribers’ from  ASR9000 BNG node but were limited by a current design limiting the possibilities? Are you currently working on your BNG design and do you need to define the best and durable solution?  

These questions come regularly from ASR 9000 BNG customers and the following article aims to provide all necessary information to shape your solution and get the most out of the platform.  

We will present internal hardware design and how it is used when it comes to BNG subscribers using QoS, the way you can affect the standard behavior and direct your subscribers to use all available resources, and finally we will discuss overall design solution that can help you reach the full potential of your ASR 9000 platform.

# Scope

- All concepts and principles that are discussed apply to IPOE and PPPOE type subscribers.
- All concepts and principles that are discussed apply to any ASR 9000 system belonging to the 3rd or 5th generation (line card or fixed chassis).
- The article considers subscriber QoS using queuing feature:
	- policy-map actions can be: priority, bandwidth remaining ratio, shaper, queue-limit, WRED etc.
	- policy-map can be flat or hierarchical; e.g. a parent policy-map to rate-limit subscriber’s overall bandwidth and a child-policy-map dedicated to QoS actions per traffic classification
	- The article mostly considers egress subscriber QoS as it is not a best practice to use ingress subscriber QoS queuing.
	- The article will mostly use Bundle-Ether interface type as examples, but the reasoning is the same for Pseudowire Headend (PW-HE) interface type.
    
# BNG QoS queuing resources default allocation

Once established, a subscriber is managed during its lifespan by a virtual interface that is dynamically created on top of the access interface you have configured. This virtual interface gathers the subscriber’s parameters: IP addressing, forwarding VRF, MTU… and QoS.  
As a reminder, you can apply QoS to subscribers using several techniques: through RADIUS (dynamic-template included), Parameterized QoS or QoS shaper parameterization.  

BNG QoS for subscriber feature is deployed on the Network Processor Unit (NPU) that handles the BNG access-interface on top of which the subscriber is established. The NPU has a unit called Traffic Manager (TM) that is responsible for allocating and managing NPU QoS queuing resources for any needs (including QoS queuing not related to BNG). Depending on the line card type and generation there can be one or two TM per NPU.  

A TM splits its queuing resources into 4 chunks: chunk0, chunk1, chunk2 and chunk3.  

Every port managed by an NPU is mapped to one of the chunks of the TM. To be comprehensive, all the sub-interfaces belonging to one port will be mapped by default to the same TM chunk.  

To illustrate the structure, here is a diagram highlighting the NPUs of the 5th generation line card A9K-8HG-FLEX-SE which has two NPUs managing eight HundredGigabitEthernet ports:

![a9k-8hg-flex-se-TM.png]({{site.baseurl}}/images/a9k-8hg-flex-se-TM.png)

We can easily retrieve this TM/chunk to port default mapping with the following commands (the command must be executed for all the NPUs of the considered line card): 

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RSP0/CPU0:BNG#show qoshal ep np 0 location 0/0/CPU0 
Sun Jun 30 10:21:40.830 CEST
TY Options argc:6 
nphal_show_chk -p 33312 ep -n 0x0 
Done

 show qoshal ep np np location node front end
 Subslot 0 Ifsubsysnum 0 NP_EP :0 State :1 Ifsub Type :0x10030 Num Ports: 1 Port Type : 100G
Port: 0
        Egress  : <span style="background-color: #ff0000">Chunk 0</span>, L1 0

Subslot 0 Ifsubsysnum 1 NP_EP :1 State :1 Ifsub Type :0x10030 Num Ports: 1 Port Type : 100G
Port: 0
        Egress  : <span style="background-color: #006600">Chunk 1</span>, L1 0

Subslot 0 Ifsubsysnum 2 NP_EP :2 State :1 Ifsub Type :0x10030 Num Ports: 1 Port Type : 100G
Port: 0
        Egress  : <span style="background-color: #003399">Chunk 2</span>, L1 0

Subslot 0 Ifsubsysnum 3 NP_EP :3 State :1 Ifsub Type :0x10030 Num Ports: 1 Port Type : 100G
Port: 0
        Egress  : <span style="background-color: #ff33ff">Chunk 3</span>, L1 0


RP/0/RSP0/CPU0:BNG#show qoshal ep np 1 location 0/0/CPU0 
Sun Jun 30 23:21:44.871 CEST
TY Options argc:6 
nphal_show_chk -p 33312 ep -n 0x1 
Done

 show qoshal ep np np location node front end
 Subslot 0 Ifsubsysnum 4 NP_EP :0 State :1 Ifsub Type :0x10030 Num Ports: 1 Port Type : 100G
Port: 0
        Egress  : <span style="background-color: #ff0000">Chunk 0</span>, L1 0

Subslot 0 Ifsubsysnum 5 NP_EP :1 State :1 Ifsub Type :0x10030 Num Ports: 1 Port Type : 100G
Port: 0
        Egress  : <span style="background-color: #006600">Chunk 1</span>, L1 0

Subslot 0 Ifsubsysnum 6 NP_EP :2 State :1 Ifsub Type :0x10030 Num Ports: 1 Port Type : 100G
Port: 0
        Egress  : <span style="background-color: #003399">Chunk 2</span>, L1 0

Subslot 0 Ifsubsysnum 7 NP_EP :3 State :1 Ifsub Type :0x10030 Num Ports: 1 Port Type : 100G
Port: 0
        Egress  : <span style="background-color: #ff33ff">Chunk 3</span>, L1 0
</code>
</pre>
</div>

Now that we can identify the chunk to port default mapping, it is important to understand that every subscriber using QoS queuing consumes one QoS hardware entity that is called L3(8Q) or L3(16Q) depending on the generation.  

A quick word about QoS entities: the ASR 9000 scheduler is implemented through several entity levels depending on how the QoS queuing is configured; port level, sub-interface level, parent/child policy-map etc.  

Each line card generation has its own specification regarding the number of L3 entities available per chunk:
- 3rd generation / Tomahawk: 8000 L3 entities
- 5th generation / LightSpeed+: 1500 L3 entities

Let’s take a practical example and consider that we use 4 access-interfaces to serve our subscribers with QoS queuing, all the 4 access-interfaces are sub-interfaces (S-VLAN) belonging to one Bundle-Ether interface of one port, port Hu0/x/0/5 in our example.  

Access sub-interfaces configuration is straightforward:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
interface Bundle-Ether1.10
 ipv4 point-to-point
 ipv6 enable
 service-policy type control subscriber BNG_PMAP
 pppoe enable bba-group BBAGROUP
 load-interval 30
 encapsulation dot1q 10
!
interface Bundle-Ether1.20
 ipv4 point-to-point
 ipv6 enable
 service-policy type control subscriber BNG_PMAP
 pppoe enable bba-group BBAGROUP
 load-interval 30
 encapsulation dot1q 20
!
interface Bundle-Ether1.30
 ipv4 point-to-point
 ipv6 enable
 service-policy type control subscriber BNG_PMAP
 pppoe enable bba-group BBAGROUP
 load-interval 30
 encapsulation dot1q 30
!
interface Bundle-Ether1.40
 ipv4 point-to-point
 ipv6 enable
 service-policy type control subscriber BNG_PMAP
 pppoe enable bba-group BBAGROUP
 load-interval 30
 encapsulation dot1q 40
!
</code>
</pre>
</div>

QoS queuing subscribers that are established on the 4 sub-interfaces will only use chunk1 of NPU1 because of the default chunk to port mapping:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RSP0/CPU0:BNG#show pppoe summary per-access-interface 
Sun Jun 30 10:40:45.830 CEST

0/RSP0/CPU0
-----------
    COMPLETE: Complete PPPoE Sessions
    INCOMPLETE: PPPoE sessions being brought up or torn down

Interface                        BBA-Group  READY   TOTAL  COMPLETE  INCOMPLETE
-------------------------------------------------------------------------------
BE1.10                  	    BNG_PMAP 	   Y     200       200           0
BE1.20                  	    BNG_PMAP 	   Y     300       300           0
BE1.30                  	    BNG_PMAP 	   Y     400       400           0
BE1.40                  	    BNG_PMAP 	   Y     600       600           0
                                             ----------------------------------
TOTAL                                           4    1500      1500           0

RP/0/RSP0/CPU0:BNG# show qoshal resource summary np 1 location 0/0/CPU0 | begin "CLIENT : QoS-EA"
Mon Jul  1 00:19:54.131 CEST
CLIENT : QoS-EA 
   Policy Instances: Ingress 0 Egress 1500  Total: 1500
    TM 0 
    Entities: (L4 level: Queues)
     Level   Chunk 0           Chunk 1           Chunk 2           Chunk 3          
     L4         0(    0/    0)10280(10280/10280)    0(    0/    0)    0(    0/    0)
     L3(8Q)     0(    0/    0) 1500( 1500/ 1500)    0(    0/    0)    0(    0/    0)
     L3(16Q)    0(    0/    0)    0(    0/    0)    0(    0/    0)    0(    0/    0)
     L2         0(    0/    0)    0(    0/    0)    0(    0/    0)    0(    0/    0)
     L1         0(    0/    0)    0(    0/    0)    0(    0/    0)    0(    0/    0)
   Policers : 3072(3072)
</code>
</pre>
</div>

