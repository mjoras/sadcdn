---
title: "Securing Ancillary Data for Communicating with Devices in the Network"
abbrev: "SADCDN"
category: info

docname: draft-joras-sadcdn-latest
submissiontype: IETF  # also: "independent", "IAB", or "IRTF"
# number: 1
date: 2023-07-05
consensus: true
v: 3
# area: AREA
# workgroup: WG Working Group
keyword:
 - next generation
 - unicorn
 - sparkling distributed ledger
#venue:
#  group: WG
#  type: Working Group
#  mail: matt.joras@gmail.com
#  arch: https://example.com/WG
#  github: USER/REPO
#  latest: https://example.com/LATEST

author:
 -
    fullname: Matt Joras
    organization: Meta Platforms, Inc.
    email: matt.joras@gmail.com

normative:

informative:


--- abstract

There is increasing need for application endpoints to communicate with devices in the network without exposing that information to on path observers.


--- middle

# Introduction
In modern mobile networks it is extremely common for policies to be applied to network flows by devices in the network. These policies are usually implemented by network vendors and enabled by mobile network operators (MNOs) to achieve certain outcomes. The two most prominent examples of this are traffic shaping and packet prioritization.

Traffic shapring in this context is a modification applied to the flow of packets to limit the achievable throughput by the flow to a given bandwidth (e.g. 2Mbps).

Packet prioritization policies are meant to prioritize certain kinds of data in the device queues over others. For example, an operator may want to employ a policy which gives queue priority to low latency video conferencing traffic over long form video playback traffic, to ensure lower latency for the more latency-sensitive user experience.

While these goals seem straightforward, and at first glance it seems like the network device can achieve them in isolation, without content endpoint cooperation there are issues that inevitably arise and pathologies which are detrimental to user experience.

# Shaping Adaptive Video Traffic
The goal of these policies are variable, but usually are motivated by limiting data usage. One goal that’s fairly common is for the operator to limit the total possible data used by customers on an “Unlimited” data plan. With these plans there are no “hard” limits (i.e. where network access ceases), rather the MNO will apply policies to specific flows such that the amount of data reaching the customer’s device for specific adaptive servers is effectively capped. These policies are targeted at flows known to carry certain kinds of data, such as video. The detection method varies, but typically the flows are identified based on the SNI in the TLS ClientHello, or similar.

Modern video playback typically employs adaptive bitrate (ABR) schemes to dynamically adjust the video quality (and thus the data rate) in response to changing network conditions. Ideally the ABR scheme should adapt the quality and converge on a bitrate sustainable by the network policer. In practice this is extremely difficult to achieve while maintaining a good user experience, due to the myriad complexities and interactions involved, such as the transport congestion control behavior, changing radio signal strength, etc.

The end goal of limiting a customer’s aggregate data usage can instead be achieved through having a content endpoint mediate the amount of data served to a given user. This capability is already present in data-heavy applications such as streaming video. For example, if a content endpoint limits a given user’s video bitrate to ~2Mbps and also limits the number of outstanding videos being streamed to that user, the overall effect on aggregate data usage is the same as if the network itself employs a policer configured to a 2Mbps data rate. Networks are able to achieve better efficiencies while still maintaining data usage limits by having the content producing endpoints limiting the data sent, rather than relying on a network device to impose an artificial limit.


# Packet Prioritization
For packet prioritization there is a different problem. While the network device may be able to make inferences about what kinds of content different packets and flows carry, it has become increasingly difficult as traffic is encrypted more holistically. Newly endemic protocols like QUIC are being used for a diverse range of traffic types, and this makes heuristics such as “all low latency traffic looks like WebRTC or RTP” untenable. Additionally, if multiple application flows are being multiplexed over a single encrypted transport, such as QUIC, the network device may want to make different prioritization decisions depending on the application contained within any given packet. 
Information Disparity
In both situations, there is an information disparity between devices in the network and the content endpoints. In both of these situations better outcomes can be achieved by explicit communication and cooperation.

In the case of a data-limiting policy, it would be advantageous for the network device to explicitly communicate the desired limits to the content endpoint so that it can “self-regulate”, and in exchange for the in-network policer to be disabled. For prioritization, it would be advantageous for the endpoint to communicate the content type of different packets so that they can be prioritized correctly.

# Out of Band vs. Inband Communication
There are generally two ways to resolve this information disparity between the content endpoints and the network: communicating additional information out of band, or inband.

Out of band communication involves the content endpoint and the MNO exchanging information in a separate context from the flow in question. There are various ways this could occur in practice, such as facilities provided by 3GPP, emerging API standards like CAMARA, or bespoke Internet API endpoints maintained by the MNO and accessed by a content endpoint. Regardless of which method is used, there are a few issues with using this form on information exchange that makes them undesirable.

The core issue though is one of association. Suppose there’s a flow that exists between an end user device and a content endpoint server on the Internet. The endpoint server has relatively little information about this user initially, mostly its basics such as the 5-tuple associated with the flow, of which the most identifying information is the IP address. In order to exchange information with the MNO about this, it has to be able to query the defined API and exchange this information. In practical terms this may range in difficulty from challenging to simply impossible. Further, the API endpoint being communicated with is often not the same entity as a network device which is applying the relevant policies. Thus even after communication is established and information is exchanged, the MNO API endpoint has the further responsibility of taking action on that information, which involves further communication within its network.

Inband communication, as the name suggests, is any mechanism by which devices in the network and the content endpoints can communicate directly. This is, in a sense, merely an extension of how all Internet Protocols as we know them today function. And indeed there are even examples of where such communication is done inband to facilitate cooperation, such as ECN marking. However to date all these systems stop short of what one might think of as a “communication channel” for exchanging rich information between the network device and a content endpoint. Such a communication mechanism has benefits over the out of band alternative, mostly in the form of simplicity for both parties. If the communication channel is established between the network device and the content endpoint directly then the relevant information can be exchanged, and acted upon, directly.

To use a concrete example, consider the case of traffic policing. Suppose that there is a content provider who, in cooperation with certain MNOs, is willing to limit the aggregate video data served to a given user, and in exchange the MNO disables the network policer for that user’s flows. The network device would identify these flows and, inband with the flow’s packets, establish a communication channel with the flows’ destination content endpoint. The network device would communicate the desired limits to the content endpoint, and the content endpoint would acknowledge the limits. The network device would then simply disable (or significantly modify) the policing configuration it otherwise would have applied.

#Securing Information Exchange
A major challenge with this inband approach in particular is how to ensure the privacy and integrity of the data being exchanged. The benefits of integrity protection are self-evident – a bad actor on the path should not be able to modify the communication such that it alters the behavior of the network or the content endpoint. Privacy is similarly important. It is not acceptable that an on-path observer should be privy to the information being exchanged between the network device and the content endpoint. Allowing this would enable a whole host of privacy vulnerabilities which are all too commonplace on the Internet today. The solution to both these problems is to encrypt the communication using a standard cryptographic protocol. Utilizing standardized cryptography also solves problems of trust and authenticity, by allowing the parties to utilize existing authentication features of cryptographic protocols.

# End User Transparency
Today end users are generally unaware of policies like shaping or prioritization being applied to their flows, especially in realtime. This is due to the fact that there is no means by which to inform them as it is happening. This information can be surfaced to the user by cooperating and exchanging rich information with the network device applying the policies, i.e. when SADCDN is being used. Consider an example where an end user’s plan has exceeded some predefined monthly quota and the network device has informed the content endpoint to put a cap on video bitrate. Since the application is the one applying this cap, it can convey that information to the user via the application user interface. 

# Proposed Solution Sketch
This proposed solution sketch first focuses on solving this problem for UDP-based protocols, such as QUIC. This is partially because of QUIC’s increasing ubiquity on the Internet for serving content of this kind, but also because the solution itself involves utilizing QUIC. Note that this ends up looking very similar to certain other schemes such as QUIC-aware proxying ({{?I-D.draft-pauly-masque-quic-proxy-06}}).

Recall that the desired goal here is for a network device to be able to, inband with a new flow of QUIC packets, establish a communication channel with the content endpoint to which those QUIC packets are destined. The key mechanism to achieve this is for the network device to establish its own QUIC connection with the same content endpoint by appending its own QUIC packets to some part of the UDP/IP packet of the original flow.

There are broadly two ways this could be done. One which seems relatively straightforward would be for the network device to modify the packet by adding on a UDP option or (newly defined) IP header, the value of which is a QUIC packet. There are issues with this approach though. Either a UDP option or an IP header could be “bleached” by other devices in the network, or not supported by the operating systems for the mobile device or content endpoint.

Another option which avoids this issue would be for the network device to modify the UDP payload of the UDP/IP packet. To achieve this the network device could encapsulate the original UDP payload within another layer, similar to what was proposed with PLUS ({{?I-D.draft-trammell-plus-spec-01}}). In this way each UDP payload would effectively contain two payloads: the original UDP payload and the payload of a QUIC packet for the channel between the network device and the content endpoint. The content endpoint would have to be able to recognize this type of packet, of course.

In either case, it is important to note the distinct advantages of coupling the packets, versus the network device sending its own packets. The most important property is that it allows guaranteeing that the end-to-end flow and the inband flow arrive at the same content endpoint. If the network device sent its own packets instead, there would have to be some mechanism ensuring that the packets are routed to the same endpoint. Another useful property is that it allows the network device to have a much simpler QUIC implementation, as it does not have to make any decisions about when and if it can send packets on its own. It makes that decision only on forwarding a UDP/IP packet.

Using this scheme a network device can initiate its own QUIC connection with the content endpoint as part of an existing UDP flow. This QUIC connection is cryptographically independent from the end-to-end UDP flow, and once established can be used as a secure communication channel between the network device and the content endpoint. Another way to think about this is that the QUIC packets used for network device to content endpoint communication are simply encrypted packet metadata associated with the end user’s flow.

# Diagrams
~~~
 Mobile Device   Packet Core Device       CAP Endpoint Server                     
     +--+          +------------+           +---------+                           
     |  |-----------------------------------|         |                           
     |  |          |            |           |         |                           
     +--+          |            |x-x-x-x-x-x|         |                           
                   +------------+           +---------+                           
                                                                                  
          -----------               x-x-x-x                                       
      e2e QUIC connection    SADCDN QUIC connection                               
~~~

In the above we can see a visualization of this idea, assuming that the end-to-end flow is a QUIC connection. These form two completely independent cryptographic contexts. Thus, only the content endpoint can securely communicate with both the network device and the mobile device. This can be used by the network device to, for example, communicate the policer configuration to the content endpoint, which can then influence the video playback to self-regulate and avoid the policing. We can also use a similar scheme to establish a channel between the mobile device and the packet core device.

Note that it would also be possible for the mobile device and the packet core device to have the secure connection, as below.

~~~
 Mobile Device   Packet Core Device       CAP Endpoint Server                     
     +--+          +------------+           +---------+                           
     |  |-----------------------------------|         |                           
     |  |x-x-x-x-x-|            |           |         |                           
     +--+          |            |           |         |                           
                   +------------+           +---------+                           
                                                                                  
          -----------               x-x-x-x                                       
      e2e QUIC connection    SADCDN QUIC connection                               
~~~

Finally, here is roughly what the scheme might look like at the packet layer. Essentially what we see is that an existing flow is appended to include the SADCDN QUIC packets. This is only seen on one side of the packet core device, the side with the established SADCDN connection. It is important to note that not every packet needs this additional information.

~~~
                    |                                                    
              Packet Core Device                                         
                    |                                                    
                    |                                                    
                    |                                                    
                    |                                                    
+-----+   +-----+   |  +-----+   +-----+   +-----+                       
|XXXXX|   |XXXXX|   |  |YYYYY|   |XXXXX|   |YYYYY|                       
|XXXXX|   |XXXXX|   |  |XXXXX|   |XXXXX|   |XXXXX|                       
+-----+   +-----+   |  |XXXXX|   +-----+   |XXXXX|                       
                    |  +-----+             +-----+                       
                    |                                                    
                                                                         
       XXXX             YYYY                                             
       e2e QUIC Data    SADCDN QUIC data                                 
~~~



# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.

