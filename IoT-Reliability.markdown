% Can the Web Connect the Internet of Things Reliably?
% Richard Clark
% November 17, 2014

# The web as a distributed system

## Characteristics

- Largely unidirectional (pull)
- Largely request/response
- Hugely popular

## What makes the web work as a distributed system?

- GET is idempotent
- Existing infrastructure
- Uniform addressing via URLs
- Familiar mental model

## What makes the web unreliable?

- Interactions aren't idempotent
- Existing infrastructure
- No concept of transactions

## When the web fails

+ Reloading as a recovery strategy
    - ...not for connected Things!
- The security model (or lack of same)
- When proxies attack
- Who gets priority

<div class="notes">
Talk about the problem of duplicated requests (e.g. in commerce), how people recover from errors (manual reloading), how this exacerbates traffic problems, etc.

Also discuss the security model, cache-related challenges, lack of traffic prioritization, etc.
</div>

# Fallacies of distributed computing

Derived from [this paper](http://www.rgoarchitects.com/Files/fallacies.pdf

1.	The network is reliable
2.	Latency is zero
3.	Bandwidth is infinite
4.	The network is secure
5.	Topology doesn't change
6.	There is one administrator
7.	Transport cost is zero
8.	The network is homogeneous

<div class="notes">
With the example of the web in mind, let's look at the ways that distributed computing could go wrong.
</div>

## "The network is reliable"

Sources of trouble:

- Overloaded application servers
- DNS failures
- Breaks in peering and other routing errors
- Trouble at the datacenter
- Local transport issues
- Mobile clients

<div class="notes">
Let's think of all the ways a simple GET can go wrong. App servers can become overwhelmed, network links can be broken for multiple reasons, you might even have trouble getting on the network locally. Mobile clients present their own issues as connections can come and go when moving (or even standing still, thanks AT&T)

How might you detect these problems? How would you correct them? (Discussion) Possible answers: ping/pong, retrying at regular intervals, CDNs and other distributed caching, etc.

Another approach is to use a messaging fabric that supports full reliable messaging (though this doens't mean you can ignore network issues!)
</div>

## "Latency is zero"

- When is a break not a break?
- TCP/IP doesn't *always* deliver in order
- (Related) Race conditions between multiple data channels

<div class="notes">
Enough latency and the other side might start timing out. This can wreak havoc in a distributed system (and often leads to cascading failures.)

TCP/IP is designed for in-order delivery *in a single session*. But take the example of a mobile client on a train (perhaps reporting entry and exit of each block), if these are individual AJAX sends they could arrive out of order if latency is inconsistent and high enough.

If you have multiple channels (e.g. a snapshot followed by a stream), there's a risk of a race condition where your snapshot can be outdated relative to the stream. (Discussion approaches to mitigating this.)

What are the possible mitigations? (Discussion) Reduce the number of calls, pass as much as possible per call, have long-lived sessions (e.g. websocket). Timestamps might help, if the data is coming from one source with a relatively consistent clock (i.e. not multiple sources as synchronization is never guaranteed.) A monotonically increasing identifier is an even better bet, especially if it can be used to detect gaps in the message sequence.
</div>

## "Bandwidth is infinite"

Might not seem like a problem, but...

- Congested networks
- Rural access & other low-bandwidth networks
- Saturating NICs, links, etc.
- Volume == latency

<div class="notes">
Bandwidth issues raise their head when you're on a network that's oversubscribed, low-bandwidth, or at its carrying capacity.

This is one of my largest indictments of REST: the overhead can overwhelm the data being sent.
</div>

## Bandwidth == latency (sometimes)

![COMET benchmark](https://webtide.com/wp-content/uploads/2011/09/http.png)
From [CometD benchmark](https://webtide.com/cometd-2-4-0-websocket-benchmarks/)

<div class="notes">
Talk about latency under conditions of network saturation.
</div>

## The same benchmark with WebSockets

![CometD Websocket benchmark](https://webtide.com/wp-content/uploads/2011/09/websocket.png)

## "The network is secure"

Classes of potential exploits:

- HTTP weaknesses (e.g. header injections)
- Denial of Service
- Man in the middle / interception / spoofing
- Weaknesses in device web stacks

<div class="notes">
Brainstorm classes of exploits. What's the likely lifetime of your devices, upgradability, risk from old devices in the field? (And what can you do when endpoints are compromised?)

Mitigations: Minimize number of connection handshakes, use of PKCS, continuous stateful connections (enable synchronized state machines on both ends), device retirement.
</div>

## "Topology doesn't change"

- Machines move, local networks change, messaging paths change
- What do you do when a node becomes detached?
- What do you do when a whole subnet becomes detached (and the problem of "split brain")

<div class="notes">
Less of an issue if you rely on DNS, but even local network issues can get you into trouble. How do you maintain connectivity when your protocols change and/or your endpoint names change (e.g. queues, topics.)

Brainstorm issues and solutions. Possible answers: protocol negotiations, dual infrastructure, monitoring at the hub to identify detached nodes (and some mechanism for following up.)
</div>

## "There is one administrator"

- Do you depend on somebody's WiFi?
- Risks from OS updates (when not embedded)
- Power & other environmental issues

<div class="notes">
</div>

## "Transport cost is zero"

- Bandwidth is not infinite (redux)
- Trade-off around adding layers / protocols / tools

<div class="notes">
Talk again about when happens when you hit NIC saturation. Also, adding a new layer to a protocol often cmoes with overhead; it's an accountable cost.

</div>

## "The network is homogeneous"

- We're not really falling for this one
- This is why open standards win (lock-in and different speeds of upgrading)

<div class="notes">
Added by James Gosling in 1988. Most architects don't fall into this trap, but beware the risks introduced by proprietary protocols and what happens when deployments don't happen across the whole network at once.
</div>

# Building more reliable distributed systems

## Idempotency

- "[A]n operation that will produce the same results if executed once or multiple times" [http://en.wikipedia.org/wiki/Idempotence#Computer_science_meaning](Wikipedia)

## Hub and spoke vs. peer to peer

- Techniques (and risks) in peer-to-peer
- Connectivity (where is the network likely to break?)
- Message forwarding via hubs

<div class="notes">
We're generally talking about a subset of distributed systems here (as opposed to a distributed database, say) where devices are forwarding results to a central location and receiving commands from there or another fixed hub. But let's talk about true peer to peer and its options, where the network is likely to be severed, and what architetectures could work.
</div>

## Resilience to network issues

- HTTP GET (e.g.) and head of line blocking
- Two Phase Commit (recipients indicate readiness, sent commit)
- Three Phase Commit (recipients indicate readiness, controller indicates intent, sent commit)


## Handling duplicate transmissions

- Monotonically increasing IDs
- *Not* timestamps!
- Multiple store and forward sources (e.g. Redis under partition)

<div class="notes">
Talk about the simple case of a client forwarding to one reader and holding the value until the eventual ack. Serial numbers as a reliable indicator (not timestamps, and explain why.) What happens when a network partition can mean a new master (e.g. Redis) and why the old master must reject writes as soon as it loses master status. (However, we also have the chance to present values back to the client for confirmation, but we always have a risk of data loss.)
</div>


# Recovery strategies

## How much consistency do you need?

- Perfectly consistent systems are *slow*
- Tolerate inconsistencies, resolve lazily

## Idempotency in action

- Can you ask for the information again?
  - e.g. single command response vs. overall state
  - Can you look at a transaction log?

## Cheap recovery

- Message brokers and the Last Value Cache
- Remembering your subscriptions

## Final thoughts

- Rolling your own CAP systems isn't recommended
- Laziness is a virtue
- So is repeating yourself
- Repeatedly

## For more information

- See the [Jepsen](http://aphyr.com/tags/jepsen) archives
- Especially [The Network is Reliable](http://aphyr.com/posts/288-the-network-is-reliable) and the [Final Thoughts](http://aphyr.com/posts/286-call-me-maybe-final-thoughts
)

# Contact information

Richard Clark
[richard.clark@kaazing.com](mailto:richard.clark@kaazing.com)
Twitter/github: rdclark
