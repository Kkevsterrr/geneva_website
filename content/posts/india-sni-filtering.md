+++
title =  "Evading SNI Filtering in India with Geneva"
lang = "en"
date = 2021-07-27
tag = []
featured_image = "indian-flag.jpeg"
summary = "In July of 2020, the Open Observatory of Network Interference (OONI) team [discovered](https://ooni.org/post/2020-tls-blocking-india/) that Indian ISPs (Airtel and Reliance Jio) had started filtering HTTPS websites using the Server Name Indication (SNI) field in TLS. A year later, we revisit Airtel's HTTPS censorship system, show how we trained Geneva against the new censorship system to discover evasion strategies, and learn more about how the censorship systems operate."
author = "Kevin Bock, Yair Fax, and Dave Levin"
+++

# Introduction

**Summary: In July of 2020, the Open Observatory of Network Interference (OONI) team [discovered](https://ooni.org/post/2020-tls-blocking-india/) that Indian ISPs (Airtel and Reliance Jio) had started filtering HTTPS websites using the Server Name Indication (SNI) field in TLS. A year later, we revisit Airtel's HTTPS censorship system, show how we trained Geneva against the new censorship system to discover evasion strategies, and learn more about how the censorship systems operate.**  

In July of 2020, the Open Observatory of Network Interference (OONI) team [discovered](https://ooni.org/post/2020-tls-blocking-india/) that Indian ISPs (Airtel and Reliance Jio) had started filtering HTTPS websites using the Server Name Indication (SNI) field in TLS. The SNI field is included in the TLS ClientHello message—in plaintext—specifying the hostname a client wants to connect to. The SNI is useful if there are multiple domains hosted on one IP address and the client wants to indicate which of those domains' certificates it wants to connect to. However, because the SNI is transmitted in plaintext, it presents an opportunity for censors to identify encrypted connections with forbidden websites.

In this article, we will:

- Detail how Airtel's censor blocks TLS connections,
- Describe a plugin to train [Geneva](http://censorship.ai) against the new censorship system, and
- Present five vulnerabilities we found in the censor and how to take advantage of them for censorship evasion

All experiments described in this article were run from vantage points that we control in Bangalore, India to VPSes we control in Iowa and Tokyo and to `example.org`, `google.com`, and `amazon.com`. Our vantage points were all within the Airtel ISP.

# How does Airtel Censor?

We find that Airtel censors forbidden connections by sending one `RST+ACK` packet in each direction—to the client and to the server. The `RST+ACK` packet has sequence and acknowledgement fields that match the `PSH+ACK` packet it's trying to censor. [I](https://geneva.cs.umd.edu/papers/geneva_ccs19.pdfhttps://geneva.cs.umd.edu/papers/geneva_ccs19.pdf)[n some of our prior work](https://geneva.cs.umd.edu/papers/geneva_ccs19.pdf), we explored Airtel's HTTP censorship system, and found that it was only active on port 80 at the time. Today, we find that the censor now runs on ports 80 (typically HTTP) and 443 (typically TLS/HTTPS): any other port is uncensored (so a website hosted on any other port could be used to evade censorship). Notably, Airtel's HTTP and HTTPS censorship systems are not bound to their respective ports: it still checks the TLS SNI field if used on port 80 and for forbidden HTTP connections on port 443.

We also find that not all IP addresses are affected by the censor. For example, we find that [`example.org`](http://example.org) and [`google.com`](http://google.com) (as well as the VPSes we control in Google Cloud) are affected, but [`amazon.com`](http://amazon.com) and our vantage points in Amazon EC2 are not affected. The rationale behind why certain IP address ranges are filtered and not others is not clear.

[Unlike the Great Firewall of China](https://geneva.cs.umd.edu/papers/geneva_ccs19.pdf), we find that Airtel's censor does not track the state of TCP connections. Simply sending a single `PSH+ACK` packet containing a TLS ClientHello with a forbidden SNI is sufficient to trigger censorship. 

# Training Geneva

{{% callout emoji="" color="blue" %}}
**Geneva (*Gen*etic *Eva*sion)** is a genetic algorithm we developed that *evolves* censorship evasion strategies against a censor. Unlike most anti-censorship systems, it does not require deployment at both ends of the connection: it runs exclusively at one side (client or server) and defeats censorship by manipulating the packet stream to confuse the censor without impacting the underlying connection. A *censorship evasion strategy* describes how that traffic should be modified. Since Geneva will be evolving these strategies, they are expressed in a domain-specific language that comprises the DNA of each strategy. (For a full rundown of Geneva's strategy DNA syntax, see [our Github page](https://github.com/kkevsterrr/geneva)).


Geneva is comprised of two main components. First, the genetic algorithm, which can evolve new ways to defeat a censorship system given an application that experiences censorship and a fitness function. Second, the strategy engine, which applies a given strategy on the fly to modify active network traffic. All of the strategies provided here can be copy & pasted and used with Geneva's engine to evade censorship.
{{% /callout %}}

To train Geneva against Airtel's new censorship system, all we need is the ability to trigger censorship and we can write a new Geneva plugin. Geneva's plugins consists of (1) the client that triggers the censor, (2) the fitness function that evaluates how well a given strategy did at evading censorship. [Our documentation](https://geneva.readthedocs.io/en/latest/extending/plugins.html) has more information on how plugins are written. 

To trigger Airtel's new censorship, we use OpenSSL to initiate a handshake with [`example.org`](http://example.org) (or an IP address we control) with an SNI of `collegehumor.com`. If the connection gets censored, OpenSSL indicates `no peer certificate available` and `SSL handshake has read 0 bytes`. If the connection doesn't get censored, OpenSSL exits with code 0. 

This plugin is also open source and available on [our Github page](https://github.com/Kkevsterrr/geneva). 

We trained Geneva over several hours in multiple experiments. In the next section, we will explore the bugs we find in Airtel's censorship system and how they can be used to evade censorship. 

# Censorship Evasion Strategies

Using Geneva, we discovered five classes of packet-manipulation strategies that successfully evade Airtel's censorship. In our 2019 [paper](https://geneva.cs.umd.edu/papers/geneva_ccs19.pdf) that introduced Geneva, we trained Geneva against Airtel's HTTP censor. Two of the strategies below are identical to strategies we found in the past, which may suggest that Airtel's HTTPS system uses a similar (or the same) middlebox to perform censorship. 

### Strategy 1: TCP Reassembly

In 2019, we found that Airtel's HTTP censor couldn't reassemble segmented TCP packets. That still holds true for Airtel's TLS censor. If we segment the initial TLS ClientHello, the censor doesn't reassemble the packet, and the rest of the connection goes uncensored. This is the simplest strategy that Geneva found, and in Geneva's syntax: `[TCP:flags:PA]-fragment{tcp:-1:True}-|`

We can also induce this strategy on the server-side. In a recent [paper](https://geneva.cs.umd.edu/papers/come-as-you-are.pdf), we explored using strategies from the server-side to evade censorship without requiring *any* deployment from the client. We can express this as a server-side strategy by having the server induce TCP segmentation in the client by setting the window of its `SYN+ACK` to a value less than 318 (the size of the TLS ClientHello). This can be done with `[TCP:flags:SA]-tamper{TCP:window:replace:227}-|`

Also, we find that it doesn't matter where we segment the TLS ClientHello; even if the entire SNI is intact, the censor still doesn't trigger on a segmented packet.

### Strategy 2: Unexpected TCP Options

The second strategy Geneva found works by including corrupt TCP options that the censor does not recognize, causing the censor to completely ignore the packet. In Geneva's syntax, we can do this with: `[TCP:flags:PA]-tamper{TCP:options-sackok:corrupt}-|`

Note that `sackok` is just one of many possible TCP options that confuse Airtel's censor. Interestingly, Geneva found this strategy in the past against Airtel's HTTP censorship system, lending additional evidence that the SNI censorship is performed by the same middlebox (or the same middlebox manufacturer) and still has not patched the issues we originally found.  

### Strategy 3: Winning the `RST+ACK` Race

When Airtel censors our connection, it sends a single `RST+ACK` to both client and server. Recall that a `RST+ACK` packet's sequence numbers must be within the server's/client's window for them to accept it (and thereby terminate the connection). Otherwise, the recipient of the `RST+ACK` will simply ignore it.

As a result, for Airtel's censorship to work properly, it must arrive at the server *before* the client's TLS ClientHello (a `PSH+ACK` packet), or else the ClientHello will advance the server's TCP window and thus the `RST+ACK` will be ignored. In other words, because Airtel sends only 1 `RST+ACK` packet, there is a race: if the client's offending ClientHello can reach the server before the censor's teardown packet, then the server will evade censorship.

In our experiments and while training Geneva, we find that the censor's `RST+ACK` *almost always loses* to our `PSH+ACK` to the server. When this occurs, we can simply drop the `RST+ACK` at the client and the connection can continue. Geneva discovered this strategy (and many variants): `\/ [TCP:flags:RA]-drop-|`

The censor could solve this problem by sending two `RST+ACK`s (one with it's sequence number as it is now and a second one advanced for the `PSH+ACK`), but Airtel does not.

### Strategy 4: Request "Cool Down"

The next strategy Geneva discovered works by triggering censorship intentionally, but in such a way that the connection is not torn down. It appears that Airtel will not censor a second forbidden request immediately after processing a first one.  

In Geneva's syntax: `[TCP:flags:PA]-duplicate(tamper{TCP:seq:corrupt}(tamper{TCP:ack:corrupt}(tamper{TCP:chksum:corrupt},),),)-|`

This strategy duplicates the TLS ClientHello packet and corrupts the sequence and acknowledgement numbers and checksum of the first packet it sends. The censor, which doesn't check checksums, sends a `RST+ACK` with bad sequence and acknowledgement numbers in both directions (to client and server). Thus, both the client and the server ignore the `RST+ACK` and the connection continues. Since Airtel does not censor the second offending packet so soon after sending the `RST+ACK`, the real forbidden message can go through unimpeded. 

Geneva discovered a second variant of this strategy: injecting a TLS ClientHello with an innocuous SNI and a corrupt checksum at the start of the connection. After processing the innocuous packet, the censorship system seems to ignore our flow for a small period of time, which allows our forbidden TLS ClientHello to go through unharmed.

### Strategy 5: Protocol Confusion

Because Airtel's censorship system now monitors for both HTTP and HTTPS connections, Geneva discovered that it can trick it into not censoring an HTTPS connection by sending an innocuous HTTP GET request (with a corrupt checksum) first. In Geneva's syntax: `[TCP:flags:PA]-duplicate(tamper{TCP:load:replace:__HTTP_REQUEST__}(tamper{TCP:chksum:corrupt},),)-|`

The censor sees that the first data-carrying packet looks like an HTTP request and, as a consequence, seems to ignore our TLS ClientHello immediately after it. We find that both innocuous `GET` or `POST` requests have this effect. This strategy seems to be a different variant of the previous strategy `Request Cool Down`: we examine this more in the next section.

# Followup Experiments

In this section, we perform further experiments to answer questions about the strategies Geneva discovered. 

### Can we use the HTTPS cool-down to evade HTTP censorship?

Injecting an innocuous HTTP `GET` request or HTTPS SNI field (or a forbidden HTTP `GET` request or HTTPS SNI field with an incorrect sequence or acknowledgement number) is sufficient to provide a 'cool-down window' within the censor and evade SNI filtering. Unfortunately, we find that we cannot directly apply the same techniques to evade Airtel's HTTP filtering. 

If we inject a forbidden HTTP `GET` request, a forbidden TLS Client Hello, or an innocuous TLS Client Hello, we can evades HTTP filtering. But we find that inserting an innocuous HTTP `GET` request does not prevent Airtel from immediately censoring a follow-up forbidden HTTP `GET` request. We're not sure why this is; we hypothesize there is some separate code path that processes innocuous HTTP `GET` requests. 

### How often does the client win the `RST+ACK` race?

To test how often the client wins the `RST+ACK` race, i.e., the TLS ClientHello reaches the server before the censor's `RST+ACK`, we ran the following experiment. We instrumented our client to drop all inbound `RST+ACK`s and issued forbidden TLS Client Hellos to a server we controlled. If the connection is not affected by the censor, we know one of two things happened: either the censor failed to attempt censorship at all or our `PSH+ACK` beat the `RST+ACK` to the server. To account for the first possibility, we run a packet capture to check if the server received a `RST+ACK`. If we receive a `RST+ACK` but the client can still communicate after dropping it, we know that our `PSH+ACK` beat the censor's `RST+ACK` to the server.

We ran this experiment once every ten seconds over the course of 2 weeks and found that our `PSH+ACK` won 85.0% of the time. 

### How often does Airtel attempt censorship?

Using the same experiment as above, we can see how often Airtel attempts censorship by simply checking if the client received a `RST+ACK`. We find that Airtel attempts censorship 96.3% of the time. We also found that Airtel's censorship failures occur almost exclusively between approximately 8:30 AM and 11:30 PM in India (03:00 and 18:00 UTC). We hypothesize that this  depends on the censor's load, and intend to explore this in the future.

# Conclusion

In this article, we investigated Airtel's new HTTPS censorship system. We used Geneva to discover five different classes of strategies for evading Airtel's censorship, and used these strategies to better understand how Airtel's censorship operates. We found that it suffers from some of the same underlying issues as their HTTP censorship infrastructure, and that it has performance issues that make it slow to react to all packets. Collectively, these results reinforce the notion that censoring connections at the scale of a nation-state introduces challenges, and that Geneva is effective at finding ways to take advantage of the challenges that censors face.

New advances in TLS promise to make SNI obsolete, including Encrypted SNI (ESNI) and Encrypted Client Hello (ECH). Despite these advances, censors have continued to invest in their SNI filtering infrastructure, and as some countries have moved to [block ESNI completely](https://geneva.cs.umd.edu/posts/china-censors-esni/esni/), studying SNI-based censorship remains as important as ever.

{{% callout color="red" emoji="" %}}

Disclaimer: Geneva’s strategies intentionally take overt action on the network to interfere with the normal operation of censors. The genetic algorithm in Geneva trains against a live adversary, so when running the genetic algorithm to evolve new censorship strategies, it will intentionally trigger censorship many times during training. Understand the risks of using Geneva in your country before trying it.

{{% /callout %}}
