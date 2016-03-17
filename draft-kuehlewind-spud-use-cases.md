---
title: Use Cases for a Substrate Protocol for User Datagrams (SPUD)
abbrev: SPUD Use Cases
docname: draft-kuehlewind-spud-use-latest
date: 2016-03-18
category: info

ipr: trust200902
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
  -
    ins: M. Kuehlewind
    name: Mirja Kuehlewind
    role: editor
    org: ETH Zurich
    email: mirja.kuehlewind@tik.ee.ethz.ch
    street: Gloriastrasse 35
    city: 8092 Zurich
    country: Switzerland
  -
    ins: B. Trammell
    name: Brian Trammell
    role: editor
    org: ETH Zurich
    email: ietf@trammell.ch
    street: Gloriastrasse 35
    city: 8092 Zurich
    country: Switzerland


informative:
  RFC1191:
  RFC1981:
  RFC4821:
  I-D.hildebrand-spud-prototype:
  I-D.trammell-spud-req:
  I-D.ietf-ippm-6man-pdm-option:


--- abstract

The Substrate Protocol for User Datagrams (SPUD) BoF session at the IETF 92 meeting in Dallas in March 2015 identified the potential need for a UDP-based encapsulation protocol to allow explicit cooperation with middleboxes while using new, encrypted transport protocols. This document summarizes the use cases discuss at the BoF and thereby proposes a structure for the description of further use cases.

--- middle

# Introduction

This document describe use cases for a common Substrate Protocol for User
Datagrams (SPUD) that could be used by an overlaying transport or application
to explicitely expose information to middleboxes or request information from
(SPUD-aware) middleboxes.

For each use case, we first describe a problem that can not be solved with
current protocols, or only solved inefficiently. We then discuss which
information should be exposed by which party to help the described problem. We
also discuss potential mechanisms to use that exposed information at
middleboxes and/or endpoints, in order to demonstrate the feasibility of using
the exposed information to the given use case. The described mechanisms are
not necessarily proposals for moving forward, nor do  they necessarily
represent the best approach for applying the exposed information,  but should
illustrate and motivate the applicability of the exposed information.

In this document we assume that there is no pre-existing trust relationship
between the communication endpoints and any middlebox on the path. Therefore
we must always assume that information that is exposed can be wrong or nobody
will actually act based on the exposed information. However, for the described
use cases there should still be a benefit, e.g if otherwise no information
would be available.

Based on each mechanism, we discuss deployment incentives of each involved
party. There must be clear incentives for each party to justify the proposed
information exposure and at best an incremental deployment strategy. Finally,
we discuss potential privacy concerns regarding the information to be exposed,
as well as potential security issues of the proposed mechanisms.



# Firewall Traversal for UDP-Encapsulated Traffic

We presume, following an analysis of requirements in {{I-D.trammell-spud-
req}}, as well as trends in transport protocol development (e.g. QUIC, the
RTCWEB data channel) that UDP encapsulation will prove a viable approach for
deploying new protocols in the Internet. This, however, leads us to a first
problem that must be solved.

## Problem Statement

UDP is often blocked by firewalls, or only enabled for a few well-known
applications (e.g. DNS, NTP). Recent measurement work has shown that somewhere
between 4% and 8% of Internet hosts may be affected by UDP impairment,
depending on the population studied. Some networks (e.g. enterprise networks
behind corporate firewalls) are far more likely to block UDP than others (e.g.
residential wireline access networks).

In addition, some network operators assume that UDP is not often used for
high-volume traffic, and is often a source of spoofing or reflected attack
traffic, and is therefore safe to block or route-limit. This assumption is
becoming less true than it once was: the volume of (good) UDP traffic is
growing, mostly due to voice and video (real-time) services (e.g. RTCWEB)
where TCP is not suitable.

Even if firewall vendors and administrators are willing to change firewall
rules to allow more diverse UDP services, it is hard to track session state
for UDP traffic. As UDP is unidirectional, it is unknown whether the receiver
is willing to accept the connection. Further there is no way to figure how
long state must be maintained once established. To efficiently establish state
along the path we need an explicit contract, as is done implicitly with TCP
today.

## Information Exposed

To maintain state in the network, it must be possible to easily assign each
packet to a session that is passing a certain network node. This state should
be bound to something beyond the five-tuple to link packets together.
In {{I-D.trammell-spud-req}}, we propose the use of identifiers for groups of
packets, called ("tubes"). This allows for differential treatment of different
packets within one five-tuple flow, presuming the application has control over
segmentation and can provide requirements on a per-tube basis. Tube IDs must
be hard to guess: a tube ID in addition to a five-tuple as an identifier,
given significant entropy in the tube ID, provides an additional assurance
that only devices along the path or devices cooperating with devices along the
path can send packets that will be recognized by middleboxes and endpoints as
valid.

Further, to maintain state, the sender must explicitly indicate the start and
end of a tube to the path, while the receiver must confirm connection
establishment. This, together with the first packet following the
confirmation, provides a guarantee of return routability; i.e. that the sender
is actually at the address it says it is. This implies all SPUD tubes must be
bidirectional, or at least support a feedback channel for this confirmation.
Even though UDP is not a bidirectional transport protocol, often services on
top of UDP are bidirectional anyway. Even if not, we only require one packet
to acknowledge a new connection. This is low overhead for this basic security
feature. This connection set-up should not impose any additional start-up
latency, so the sender must be also able to send payload data in the first
packet.

If a firewall blocks a SPUD packet, it can be beneficial for the sender to
know why the packet was blocked. Therefore a SPUD-aware middlebox should be
able to send error messages. Such an error message can either be sent directly
to the sender itself, or alternatively to the receiver that can decide to
forward the error message to a sender or not.

## Mechanism

A firewall or middlebox can use the tube ID as an identifier for its session
state information. If the tube ID is large enough it will be hard for a non-
eavesdropping attacker to guess the ID.

If a firewall receives a SPUD message that signals the start of a connection,
it can decide to establish new state for this tube. Alternatively, it can also
forward the packet to the receiver and wait if the connection is wanted before
establishing state. To not require forwarding of unknown payload, a firewall
might want to forward the initial SPUD packet without payload and only send
the full packet if the connection has be accepted by the receiver.

The firewall must still maintain a timer to delete the state of a tube if no
packets were received for a while. However, if a end signal is received the
firewall can remove the state information faster.

If a firewall receives a SPUD message which does not indicate the start of a
new tube and no state is available for this tube, it may decide to block the
traffic. This can happen if the state has already timed out or if the traffic
was rerouted. In addition a firewall may send an error message to the sender
or the receiver indicating that no state information is available. If the
sender receives such a message it can resend a start signal (potentially
together with other tube state information) and continue its transmission.

## Deployment Incentives

The ability to use existing firewall management best practices with new
transport services over SPUD is necessary to ensure the deployability of SPUD.
In today's Internet, application developers really only have two choices for
transport protocols: TCP, or transports implemented at the application layer
and encapsulated over UDP. SPUD provides a common shim layer for the second
case, and the firewall traversal facility it provides makes these transports more likely to deploy.

It is not expected that the information provided by SPUD will enable all
generic UDP-encapsulated transports to safely pass firewalls. However, it does
make state handling easier for new services that a firewall administrator is
willing to allow.

## Security, Privacy, and Trust

The tube ID is scoped to the five-tuple. While this
makes the tube ID useless for session mobility, it does mean that the valid ID
space is sufficiently sparse to maintain the "hard to guess" property, and
prevents tube IDs from being misused to track flows from the same endpoint
across multiple addresses. This limitation may need further discussion.

By providing information about connection setup, SPUD exposes information
equivalent to that available in the TCP header. It makes connection lifetime
information explicit and accessible without specific higher-layer/application-
level knowledge.



# On-Path State Lifetime Discovery and Management

Once the problem of connection setup is solved, the problem arises of managing the lifetime of state associated with that connection at various devices along the path: NAT and stateful firewall state timeouts are a common cause of connectivity issues in the Internet.

## Problem Statement

Devices along the path that must keep state in order to function cannot assume
that signals tearing down a connection are provided reliably. This is also the
case for current TCP traffic. Therefore, all stateful on-path devices must
implement a mechanism to remove the state if no traffic is seen for a given
flow or tube for a while. Usually this is implemented by maintaining a timeout
since the last observed packet.

If the timeouts are set too low, on-path state might be discarded while the
endpoint connection is still alive; in the case of firewalls and NATs, this
can lead to unreliable connectivity. The common solution to this problem is
for applications or transport protocols that do not have any productive
traffic to send to send "heartbeat" or "keep-alive" packets to reset the state
timeout along the path. However, since the minimum timeout along the path is
unknown to the endpoint, implementers of transport and application . A default
value of 150ms is commonly used today. This represents a fairly rapid
generation of nonproductive traffic, and is especially onerous on battery-
powered mobile devices, which must wake up radios and switch to a higher-power
mode to transmit these nonproductive packets, leading to suboptimal power
usage and shorter battery life.

## Information Exposed

SPUD can be used to request that SPUD-aware middleboxes along the path expose
their minimum state timeout value. Here, the sending endpoint sends a
"accumulate minimum timeout" request along with some scratch space for
middleboxes to place their timeout information in. Each middlebox inspects
this value, and writes its own timeout only if lower than the present value.

Applications may also send a "timeout proposal" to devices along the path
using a SPUD declaration that a given tube will send a packet at least once
per interval, and if no packet is seen within that interval, it is safe to
tear down state.

These two declarations may be used together, with middleboxes willing to use
the application's value setting their timeouts on a per-tube basis, or
exposing a lower timeout value to allow the application to adjust.

## Mechanism

If a SPUD-aware middlebox that uses a timeout to clean up per-tube state
receives a SPUD minimum timeout accumulation, it should expose its own timeout
value if smaller than the one already given. Alternatively, if a value is
already given, it might decide to use the given value as timeout for the state
information of this tube. An endpoint receiving an accumulated minimum timeout
should send it back to its remote peer via a feedback channel. Timeouts on each
direction of a connection between two endpoints may, of course, be different,
and are handled separately by this mechanism.

If a SPUD-aware middlebox that uses a timeout to clean up per-tube state
receives a timeout proposal, it should set its timeout accordingly, subject to
its own policy and configuration.

These mechanisms are of course completely advisory: there may be non-SPUD
aware middleboxes on path which will ignore any proposed timeout and not
expose their timeout information, and middleboxes must be configured with
maximum timeout proposal they will accept in order to defend against state
exhaustion attacks.

Endpoints must therefore be combine the use of these signals endpoint with a
dynamic timeout discovery and adaptation mechanism, which uses the signals to
set initial guesses as to the path timeout.

## Deployment Incentives

Initially, if not widely deployed, there will be not much benefit to using
this extension.

However, we can assume that there are usually only a small number of
middleboxes on a given network path that hold per-tube state information.
Endpoints have an incentive to request minimum timeout and to propose timeouts
to improve convergence time for dynamic timeout adaptation mechanisms, and
middleboxes have an incentive to cooperate to improve reliability of
connections as well as state management. It is therefore likely that if
information is exposed by a middlebox, this information is correct and can be
used.

The more SPUD gets deployed, the more often endpoints will be able to set the
heartbeat interval correctly. This will reduce the amount of unproductive
traffic as well as the number of reconnections that cause additional latency.

Likewise, SPUD-aware middleboxes that expose  timeout information are able to
handle timeouts more flexibly, e.g. announcing lower timeout values when they
have less space available for new state. Further if an endpoint announces a
low pre-set value because the endpoint knows that it will only have short idle
periods, the timeout interval could be reduced.


## Security, Privacy, and Trust

Timeout proposals increase the risk of state exhaustion attacks for SPUD-aware
middleboxes that naively follow them. Likewise, accumulated minimum timeouts
could be used by malicious middleboxes to induce floods of useless heartbeat
traffic along the path, and/or exhaust resources on endpoints that naively
follow them. All timeout proposals and minimum timeouts must therefore be
inputs to a dynamic timeout selection process, both at endpoints and on-path
devices, which use these signals as hints but clamp their timeouts to sane values set by local policy.

While device timeout and heartbeat interval are generally not linked to
privacy-sensitive information, a timeout proposal may add a number of bits of
entropy to an endpoint's unique fingerprint. It is therefore advisable to
suggest a small number of useful timeout proposals, in order to reduce this
value's contribution to an endpoint fingerprint.

# Path MTU Discovery

Similar to the state timeout problem is the Path MTU problem: differing MTUs on different devices along the path can lead to fragmentation or connectivity issues. This problem is made worse by the increasing proliferation of tunnels in the Internet, which reduce the MTU by the amount required for tunnel headers. 

## Problem Statement

In order to efficiently send packets along a path end to end, they must be
sized to fit in the MTU of the "narrowest" link along the path. Algorithms for
path MTU discovery have been defined and standardized for a quarter century,
in {{RFC1191}} for IPv4 and {{RFC1981}} for IPv6, but they are not often
implemented due in part to widespread impairment of ICMP. Packetization Layer
Path MTU Discovery {{RFC4821}} (PLPMTUD) is a more recent attempt to solve the problem,
which has the advantage of being transport-protocol independent and functional
without ICMP feedback. SPUD, as a shim between UDP and superstrate transport
protocols, is at the right place in the stack to implement PLPMTUD, and
explicit cooperation can enhance its operation.

## Information Exposed

SPUD can be used to request that SPUD-aware middleboxes along the path expose
their next-hop path MTU value. Here, the sending endpoint sends a "accumulate
minimum MTU" request along with some scratch space for middleboxes to place
the next-hop MTU for the given tube. Each middlebox inspects this value, and
writes its own next-hop MTU only if lower than the present value.

A SPUD-aware middlebox that receives a packet that is too big for the next-hop MTU can send back a signal associated with the tube directly to the sender, including the next-hop MTU.

## Mechanism

PLPMTUD functions by dynamically increasing the size of packets sent, and
reacting to the loss of the first "too large" packet as an MTU reduction
signal, instead of a congestion signal. This must be implemented in
cooperation with the superstrate transport protocol, as it is responsible for
how non-MTU-related loss is treated.

When an endpoint receives an accumulated minimum MTU, it should should send it
back to its remote peer via a feedback channel. The minimum of this value and
any direct next-hop MTU signals received from SPUD-aware middleboxes can be
used as a hint to the sender's PLPMTUD process, as a likely upper bound for
path MTU associated with a tube.

## Deployment Incentives

As with state lifetime discovery, these signals are of little initial utility
to endpoints before SPUD-aware middleboxes are deployed. However, SPUD-aware
middleboxes that sit at potential MTU breakpoints along a path, either those
which terminate tunnels or bridge networks with two different link types, have
an incentive to improve reliability by responding to accumulation requests and
sending next-hop MTU messages to SPUD-aware endpoints.

## Security, Privacy, and Trust

As with state lifetime discovery, Minimum MTU and next-hop MTU signals could
be used by malicious middleboxes to set the endpoint's maximum packet size to
inefficiently small sizes, if the endpoint follows them naively. For that
reason, endpoints should use this information only as hints to improve the
operation of PLPMTUD, and may probe above the value derived from the SPUD-
supplied information when deemed appropriate by endpoint policy or transport
protocol requirements.




# Low-Latency Service

## Problem Statement

Networks are often optimized for low loss rates and high throughput by
providing large buffers that can absorb traffic spikes or rate variations and
always holding enough data to keep the link full. This is beneficial for
applications like high-priority bulk transfer, where only the total transfer
time is of interest. (High volume) interactive application, such as video
calls, however, have very different requirements. Usually these application
can tolerate high(er) loss rates, as they anyway cannot wait for missing data
to be retransmitted, while having hard latency requirements necessary to make
their service work.

Large network buffers may induce high queuing delays due to greedy cross
traffic using loss-based congestion control that periodically fills the
buffer. In loss-based congestion control the sending rate is periodically
increased until a loss is observed to probe for available bandwidth.
Unfortunately, the queuing delay that is indices by this probing can downgrade
the quality of experience for competing interactive applications or even make
them simply unusable. Further, to co-exist with greedy flows that use loss-
based congestion control, one has to react based on the same feedback signal
(loss) and implement about the same aggressiveness than these competing flows.

## Information Exposed

While large buffers that are able to absorb traffic spikes that are often
induced by short bursts are beneficial for some applications, the queuing
delay that might be induced by these large buffers is very harmful to other
applications. We therefore propose an explicit indication of loss- vs.
latency-sensitivity per SPUD tube. This indication does not prioritize one
kind of traffic over the other: while loss-sensitive traffic might face larger
buffer delay but lower loss rate, latency-sensitive traffic has to make
exactly the opposite tradeoff.

Further, an application can indicate a maximum acceptable single-hop queueing
delay per tube, expressed in milliseconds. While this mechanism does not
guarantee that sent packets will experience less than the requested delay due
to queueing delay, it can significantly reduce the amount of traffic uselessly
sitting in queues, since at any given instance only a small number of queues
along a path (usually only zero or one) will be full.

## Mechanism

A middlebox may use the loss-/latency-sensitive signal to assign packet to the
appropriate service if different services are implemented at this middlebox.
Today's traffic, that does not indicate a low loss or low latency preference,
would still be assigned to today's best-effort service, while a new low
latency service would be introduced in addition.

The simplest implementation of such a low latency service (without disturbing
existing traffic) is to manage traffic with the latency-sensitive flag set in
a separate queue. This queue either, in itself, provides only a short buffer
which induces a hard limit for the maximum (per-queue) delay or uses an AQM
(such as PIE/ CoDel) that is configured to keep the queuing delay low.

In such a two-queue system the network provider must decides about bandwidth
sharing between both services, and might or might not expose this information.
Initially there will only be a few flows that indicate low latency preference.
Therefore at the beginning this service might have a low maximum bandwidth
share assigned in the scheduler. However, the sharing ratio should be adopted
to the traffic load/number of flows in each service class over time. This can
be done manually by a network administrator or in an automated way.

Applications and endpoints setting the latency sensitivity flag on a tube must be prepared to experience relatively higher loss rates on that tube, and might use techniques such as Forward Error Correction (FEC) to cope with these losses.

If in addition the maximum per-hop delay is indicated by the sender, a SPUD-
aware router might  drop any packet which would be placed in a queue that has
more than the maximum single-hop delay at that point in time before queue
admission. Thereby the overall congestion can be reduced early instead of
withdrawing the packet at the receiver after it has blocked network resources
for other traffic. Alternatively, a SPUD-aware node might only remove the
payload and add a SPUD error message, to report what the problem is.

An endpoint indicating the maximum per-hop delay must be aware that is might face higher loss rates under congestion than competing traffic on the same bottleneck. Especially, packets might be dropped due to the maximium per-hop delay indication before any congestion notification is given to any other competing flows on the same bottleneck. This should considered in the congestion reaction as any loss should be consider as a sign for congestion.

## Deployment Incentives

Application developers go to a great deal of effort to make latency-sensitive
traffic work over today's Internet. However, if large delays are induced by
the network, an application at the endpoint cannot do much. Therefore
applications can benefit from further support by the network.

Network operators have already realized a need to better support low latency
services. However, they want to avoid any service degradation for existing
traffic as well as risking stability due to large configuration changes.
Introducing an additional service for latency-sensitive traffic that can exist
in parallel to today's network service (or potentially fully replace today's
service at some point in future...) helps this problem.

## Security, Privacy, and Trust

An application does not benefit from wronly indicating loss- or latency-
sensitivity as it has to make a tradeoff between low loss and potential high
delay or low delay and potential high loss. Therefore there is no incentive
for lying. A simple classification of traffic in loss-sensitive and latency-
sensitive does not expose privacy-critical information about the user's
behavior.

# Reordering Sensitive Services

## Problem Statement

TCP's fast retransmit mechanism interprets the reception of three duplicated acknowledgement (where the acknowledgement number is the same than in the previous acknowledgement) as a signal for loss detection. However, a missing packet in the squence number space must not always be lost. Simple reordering where one packet takes a longer path than (at least three) subsequent packets can have the same effect.

In addition in TCP, loss is an implicit signal for network congestion. Therefore the reception of three duplicated acknowledgement will cause a TCP sender to reduce its sending rate. To avoid unnecessary performance decreases, today's in-network mechanisms usually aim to avoid reordering. However, this complicates these mechanism significantly and usually requires per-flow state, e.g. in case of Equal Cost Multipath (ECMP) routing where a hash of the 5 tuple would need to be mapped to the right path.

Even though the majority of traffic in the Internet is still TCP, it is likely that new protocols will be design such that they are (more) robust to reordering. Further with an increasing deployment of ECN, even TCP's congestion control reaction based on duplicated acknowledgements could be relaxed (e.g. by reducing the sending rate gradually depending on the number of lost packets).

However, as middlebox can not know if a certain traffic flow is sensitive to reordering or not, they have to treat all traffic as equally and try to always avoid reordering. (This does not only complicate these mechanism but might also block the deployment of new services.)

## Information Exposed

Reordering-sensitivity is a per tube signal (as reordering can only happen with a flow multiple packets). However, to avoid state in middlebox, it would be beneficial to have a reordering-sensitive flag in each packet.

A transport should set the bit if it is not sensitive to reordering, e.g. if it uses a more advance mechanism (than duplicated acknowledgement) for loss detection, or if the congestion control reaction to this signal imposes only a small performances penalty, or if the flow is short enough that it will not impact its performance.

## Mechanism

A middlebox that implement an in-network function that could lead to varying end-to-end delay and reordering (as packets might overtake each other on different paths or within the network device), do not need to perform any additional action if the reordering-sensitivity flag is not set. However, if the flag is set, the middlebox should avoid reordering by e.g. holding per-tube state and make sure that all packets belonging to the same tube will not be re-ordered.

## Deployment Incentives

Today by default middlebox assume that all traffic is reordering-sensitive which complicates certain in-network mechanism or might also block the deployment of new services. If a middlebox would know that certain traffic is not reordering-sensitive, it could reduce state, speed-up processing, or even implement new services.

Applications that are not loss-sensitive (because they e.g. uses FEC) usually are also not reordering-sensitive. At the same time these application are often sensitive to latency. If the transport handles reordering appropriately and signal this semantic information to the network, the appropriate network treatment can likely also result in lower end-to-end or at least enables the network device to impose any additional delay (e.g. to set up state) on these packets.

## Security, Privacy, and Trust

No trust relationship is needed as the provided information do not results in a preferential treatment. Only transport semantics are exposed that to not contain any private information. No security threats are known.

# Application-Limited Flows

## Problem Statement

Today, there are a large number of flows that are mostly application-limited,
where the application can adapt this limit to changing traffic conditions. An
example is unicast streaming video where the coding rate can be adapted based
on detected congestion or changing link characteristics. This adaptation is
difficult, since cross-traffic (much of which uses TCP congestion control)
will often probe for available bandwidth more aggressively than the
application's control loop. Further complicating the situation is the fact
that rate adaptation may have negative effects on the user's quality of
experience, and should therefore be done infrequently.

## Information Exposed

With SPUD, the sender can provide an explicit indication of the maximum data
rate that the current encoding needs. This can provide useful information to
the bottleneck to decide how to correctly treat the corresponding tube, e.g.
setting a rate limit or scheduling weight if served from its own queue.

Further, a network node that imposes rate shaping could expose the rate limit
to the sender if requested. This would help the sender to choose the right
encoding and simplifies probing. If the rate limited is changed the network
node might want to signal this change without being requested for it.

In addition, both the endpoint as well as a middlebox could announce sudden
changes in bandwidth demand/offer. While for the endpoint it might be most
important to indicate that the bandwidth demand has increased, a middlebox
could indicate if more bandwidth is (currently) available. Note that this
information should only be indicated if the network node was previously the
bottleneck/the out-going link is fully loaded. Further, if the information
that bandwidth is available is provided to multiple endpoints at the same
time, there is a higher risk of overloading the network as all endpoints might
increase their rate at the same time.

[Editor's note: Should a middlebox even indicate how much capacity is available.. or 1/n of the available capacity if indicated to n endpoints? But there might be a new bottleneck now...]

## Mechanism

If the maximum sending rate of a flow is exposed this information could be
used to make routing decision, if e.g. two paths are available that have
different link capacity and average load characteristics.

Further, a network nodes, that receives an indication of the maximum rate
limit for a certain tube, might decide to threat this flow in an own queue and
prioritize this flow in order to keep the delay low as long as the indicated
rate limit is not exceeded. This should only be done if there is sufficient
capacity on the link (the average load over a previous time period has be low
enough to serve an additional maximum traffic load as indicated by the rate
limit) or the flow is known to have priority, e.g. based on additional out-of-
band signaling. If the link, however, is currently congested, a middlebox
might choose to ignore this information or indicate a lower rate limit.

If a network node indicates rate shaping, this information can be used by the
sender to choose its current data/coding rate appropriately. However, a sender
should still implement a mechanism to probe ifor available bandwidth to verify
the provided information. As a certain rate limit is expected the sender
should probe carefully around this rate.

A network node might further indicate a different/lower rate limit during the
transmission. However, in this case, it might be easy for an attacker to send
a wrong rate limit, therefore an endpoint should not change its data rate
immediately, but might be prepared to see higher losses rates instead.

If a sender receives an indication that more bandwidth is available it should
not just switch to a higher rate but probe carefully. Therefore it might step-
wise increase its coding rate or first add additional FEC information which
will increase the traffic rate on the link and at the same time provide
additional protection as soon as the new capacity limit is reached.

A network node that receives an indication that a flow will increase its rate
abruptly, might prioritize this flow for a certain (short) time to enable a
smoother transition. [Editor's node: Need to figure out if high loss/delay
when the coding rate is increased is actually a problem and if so further
evaluate if short-term prioritization helps.]

## Deployment Incentives

By indicating a maximum sending rate a network operator might be able to
better handle/schedule the current traffic. Therefore the network operator
might be willing to support these kind of flows explicitly by trying to serve
the flow with the requested rate. This can benefit the service quality and
increase the user's satisfaction with the provided network service.

If the maximum sending rate is known by the application, the application might
be willing to expose this information if there is a chance that the network
will try to support this flow by providing sufficient capacity.

Currently application have no good indication when to change their coding
rate. Especially, increasing the rate is hard. Further, it should be avoided
to change the rate (forth and back) too often. An indication if and how much
bandwidth is available, is therefore helpful for the application and can
simplify probing (even though there will still and always be an additional
control loop needed to react to congestion and for probing).

## Security, Privacy, and Trust

[Editor's note: is there an attack possible by indicating a low limit (from or to the application)? Note, that the application should not rely on this information and still probe for more capacity (if needed) and react to congestion!]

# Priority Multiplexing

## Problem Statement

Many services require multiple parallel transmissions to transfer different
kinds of data which usually have a clear priority between each other. One
example is WebRTC where the audio is most important and should be higher
prioritized than the video, while control traffic might have the lowest
priority. Further, some packets within one flow might be more important than
others within the same flow/tube, e.g. such as I-frames in video
transmissions. However, today a network will treat all packets the same in
case of congestion and might e.g. drop audio packets while video and control
traffic are still transmitted.

## Information Exposed

A SPUD sender may indicate a lower priority relative to another tube that is
used in the same 5-tuple.

Similarly, a lower packet priority within one flow/tube could be indicated to
give one packet a low priority than other packets with the same tube ID. This
information can be used to preferentially drop less inportant packets e.g.
carrying information that could be recovered by FEC or where missing data can
be easily concealed. 

Further, with a stronger integration of codec and transport technology SPUD
could even indicate more even finer grained priority levels to provide
automatic graceful degradation of service within the network itself.

[Editor's note: do we want to also provide per-packet information over spud? Or would all lower priority packets of one flow simply below to a different tube? In this case can we send a SPUD start message with more than on tube ID?]

## Mechanism

Preferential dropping can be implemented by a router queue in case packets
need to be dropped due to congestion. In this case the router might not drop
the incoming packet but look for a packet with the same tube ID that is
already in the queue and has a lower priority than to actual packet that
should have been dropped.

Note that a middlebox should only drop a different packet if there is
currently a lower priority packet in the queue, because it otherwise does not
know whether it will every see a lower priority packet for this flow. This
could cause unfairness issues. Therefore a middlebox might need to hold
additional state, e.g. keeping position of the last low priority packet of
each tube in a separate table. The chance that a low priority packet of the
same or corresponding tube currently sits in the queue, is lower the smaller
the buffer is. Therefore for low-latency, real-time services, there is a
tradeoff.

Alternatively, the middlebox might queue the lower priority traffic in a
different queue. Using a different queue might be suitable for lower flow
priority but should not be used for lower priority packets within the same
flow as this can also lead to other issues such as high reordering. Further,
using a lower priority queue will not only give higher priority to the traffic
belong to the same service/sender but also to all other competing flows. This
is usually not the intention.

[Editor's note: Does it makes sense to, in addition, rate-limit the higher prirority flows to their current rate to make sure that the bottleneck is not further overloaded...?]

If a sender has indicated lower priority to certain tubes and only experiences
losses/congestion for the lower priority tubes, the sender should still not
increase its sending for the higher priority tube and might even consider to
decrease the sending rate for the higher prioroty tubes as well. Potentially a
(delay-based) mechanism for shared bottleneck detection should be used to
ensure that all transmissions actually share the same bottleneck.

## Deployment Incentives

[Editor's note: similar as above -> support of interactive services increases costumer satisfaction...]

## Security, Privacy, and Trust

As only lower priority should be indicated, it is harder to use this information for an attack.

[Editor's note: Do not really see any trust or privacy concerns here...?]

# In-Band Measurement {#meas}

## Problem Statement

The current Internet protocol stack has very limited facilities for network
measurement and diagnostics. The only explicit measurement feature built into
the stack is ICMP Echo ("ping"). In the meantime, the Internet measurement
community has defined many inference- and assumption-based approaches for
getting better information out of the network: traceroute and BGP looking
glasses for topology information, TCP sequence number and TCP timestamp based
approaches for latency and loss estimation, and so on. Each of these uses
values placed on the wire for the internal use of the protocol, not for
measurement purposes, and do not necessarily apply to the deployment of new
protocols or changes to the use of those values by protocol implementations.
Approaches involving the encryption of transport protocol and application
headers (indeed, including that the authors advance in {{I-D.trammell-spud-
req}}) will break most of these, as well.

Replacing the information used for measurement with values defined explicitly
to be used for measurement in a transport protocol independent way allows
explicit endpoint control of measurability and measurement overhead.

We note that current work in IPPM {{I-D.ietf-ippm-6man-pdm-option}} proposes a
roughly equivalent, IPv6-only, kernel-implementation-only facility.

## Information Exposed

The "big five" metrics -- latency, loss, jitter, data rate / goodput, and
reordering -- can be measured using a relatively simple set of primitives.
Packet receipt acknowledgment using a cumulative nonce echo allows
both endpoint and on-path measurement of loss and reordering as well as
goodput (when combined with layer 3 packet length headers). A timestamp echo
facility, analogous to TCP's timestamp option but using an explicitly defined,
constant-rate clock and exposure of local delta (time between receipt and
subsequent transmission). 

The cumulative nonce echo consists of two values: a number identifying a given
packet (nonce), which also identifies all retransmissions of the packet, and a
number which is the sum of all packet identifiers received from the remote
endpoint (echo), modulo the maximum value of the echo field. Nonces need not
be sequential, or even monotonic, but two packets with the same nonce should
not be simultaneously in flight. These are exposed on a per-packet basis, but
need not appear on every packet in the tube or flow, with the caveat that
lower sampling rates lead to lower sensitivity.

The timestamp echo consists of three values: The time in terms of ticks of a
constant rate clock that a packet is sent, the echo of the last such timestamp
received from the remote endpoint, and the number of ticks of the sender's
clock between the receipt of the last timestamp from the remote endpoint and
the transmission of the packet containing the echo. This last delta value is
the missing link in TCP sequence number based and timestamp option based
latency estimation.

The information exposed is roughly equivalent than that currently exposed by
TCP as a side effect of its operation, but defined such that they are
explicitly useful for measurement, useful regardless of transport protocol,
and such that information exposure is in the explicit control of the endpoint
(when the superstrate transport protocol's headers are encrypted).

## Mechanism

The nonce and timestamp echo information, emitted as per-packet signals in the
SPUD header, can be used by any device which can see it to estimate
performance metrics on a per-tube basis. This includes both remote endpoints,
as well as passive performance measurement devices colocated with network
gateways.

## Deployment Incentives

Initial deployment of this facility is most likely in closed networks such as
enterprise data centers, where a single administrative entity owns the network
and the endpoints, can control which flows and tubes are annotated with
measurement information, and can benefit from the additional insight given
during network troubleshooting by explicit measurement headers. Once the
facility is deployed in SPUD-aware endpoints, it can also be used for inter-
network and cross-Internet performance measurement and debugging.

## Security, Privacy, and Trust

The cumulative nonce and timestamp echos leak no more information about the
traffic than the TCP header does. Indeed, since the cumulative nonce does not
include sequence number information or other protocol-internal information, it
allows passive measurement of loss and latency without giving measurement
devices access to information they could use to spoof valid packets within a
transport layer connection. 

In order to prevent middleboxes from modifying measurement-relevant
information, these per-packet signals will need to be integrity protected by
SPUD.

Performance measurement boxes at gateways which observe and aggregate these
signals will necessarily need to trust their accuracy, but can verify their
plausibility by calculating nonce sums and synchronizing timing clocks.


# IANA Considerations

This document has no actions for IANA.

# Security Considerations

Security and privacy considerations for each use case are given in the
corresponding Security, Privacy, and Trust subsection.

# Acknowledgments

This document grew in part out of discussions of initial use cases for
middlebox cooperation at the IAB SEMI Workshop and the IETF 92 SPUD BoF;
thanks to the participants. {{meas}} is based in part on discussions and
ongoing work with Mark Allman and Rob Beverly.

This work is supported by the European Commission under Horizon 2020 grant
agreement no. 688421 Measurement and Architecture for a Middleboxed Internet
(MAMI), and by the Swiss State Secretariat for Education, Research, and
Innovation under contract no. 15.0268. This support does not imply
endorsement.


