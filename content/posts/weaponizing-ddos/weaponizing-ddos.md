---
title: "Weaponizing Middleboxes for TCP Reflected Amplification"
shorttitle: "Weaponizing Censors for DDoS"
featured_image: "/amplify.jpg"
show_particles: false
date: 2021-08-12T03:53:38-05:00
description: "Censors pose a threat to the entire Internet."
summary: "Censors pose an even greater threat to the Internet than previously understood. We demonstrate an off-path attack that exploits _residual censorship_, a feature by which a censor continues blocking traffic between two end-hosts for some time after a censorship event. Our attack sends spoofed packets with censored content, keeping two victim end-hosts separated by a censor from being able to communicate with one another. This attack allows anyone to weaponize censorship infrastructure to perform their own blocking."
draft: false
aliases:
    - /weaponizing
    - /weaponizing/

---

This work is presented at [USENIX Security 2021](https://www.usenix.org/conference/usenixsecurity21/presentation/bock) and received a Distinguished Paper Award.

**Summary:**

- We discover a new way that attackers could launch reflected denial of service (DoS) amplification attacks over TCP by abusing middleboxes and censorship infrastructure. These attacks can produce orders of magnitude more amplification than existing UDP-based attacks.
- This is the first reflected amplification attack over TCP that goes beyond sending `SYN` packets and the first HTTP-based reflected amplification attack.
- We found multiple types of middlebox misconfiguration in the wild that can lead to technically *infinite* *amplification* for the attacker: by sending a single packet, the attacker can initiate an *endless stream of packets* to the victim.

**Collectively, our results show that censorship infrastructure poses a greater threat to the broader Internet than previously understood.** 

See our full paper [here](https://geneva.cs.umd.edu/papers/usenix-weaponizing-ddos.pdf), our open source code [here](https://github.com/breakerspace/weaponizing-censors), or you can jump to the [FAQ section](#faq) below. 

{{% callout emoji="" color="blue" %}}

This is not the only way we've discovered that censors could be weaponized: see the post about our work in WOOT 2021 "[Your Censor is My Censor: Weaponizing Censorship Infrastructure for Availability Attacks](/posts/weaponizing-availability/weaponizing-availability/)".

{{% /callout %}}



## What's reflected amplification?

Reflected amplification attacks are a powerful tool in the arsenal of a DoS attacker. An attacker spoofs a request from a victim to an open server (e.g. open DNS resolver), and the server responds to the victim. If the response is larger than the spoofed request, the server effectively *amplifies* the attacker's bandwidth in the DoS attack:

{{< figure src="/weaponizing/udp.gif" >}}

## Weaponizing Middleboxes

Most DoS amplifications today are UDP-based. The reason for this is that TCP requires a 3-way handshake that complicates spoofing attacks. Every TCP connection starts with the client sending a `SYN` packet, the server responds with a `SYN+ACK`, and the client completes the handshake with an `ACK` packet.  The 3-way handshake  protects TCP applications from being amplifiers because if an attacker sends a `SYN` packet with a spoofed source IP address, the `SYN+ACK` will go to the victim, and the attacker never learns critical information contained in the `SYN+ACK` needed to complete the 3-way handshake. Without receiving the `SYN+ACK`, the attacker can't make valid requests on behalf of the victim.

{{< figure src="/weaponizing/udpvtcp.gif" >}}

The 3-way handshake is effective at preventing amplification for TCP-compliant hosts. But in this work, we discover a large number of **network middleboxes** do not conform to the TCP standard, and can be abused to perform attacks. In particular, we find many censorship middleboxes will respond to spoofed censored requests with large block pages, even if there is no valid TCP connection or handshake. These middleboxes can be weaponized to conduct DoS amplification attacks.

Middleboxes are often not TCP-compliant by design: many middleboxes attempt handle asymmetric routing, where the middlebox can only see one direction of packets in a connection (e.g. client to server). But this feature opens them to attack: if middleboxes inject content based only on one side of the connection, an attacker can spoof one side of a TCP 3-way handshake, and convince the middlebox there is a valid connection.

{{< figure src="/weaponizing/weaponizing_middleboxes.gif" >}}

This leaves us with some questions: what is the best way to trigger these middleboxes, and what kinds of amplification factors can we get from them? 

## Discovering Amplifying Middleboxes

Our goal is to discover a sequence of packets that an attacker can send to trick a middlebox into injecting a response without completing a real 3-way handshake. 

Note that this goal is not compliant with TCP. We are taking advantage of weaknesses in *implementation,* not in the design of the TCP protocol itself. This means it's not sufficient to study the TCP protocol alone - we must study real middlebox TCP implementations. This poses a challenge: there are too many kinds of middleboxes around the world for us to purchase, and even if we could, the middleboxes that power nation-state censorship infrastructure are usually not for sale.

Instead, we used our tool [Geneva](https://geneva.cs.umd.edu/about/) to study censoring middleboxes in the wild. 

To find middleboxes to study, we used the public data released from CensoredPlanet's Quack tool. Quack is a scanner that finds IP addresses with a censoring middlebox on their path. We used this data to identify 184 sample middleboxes located around the world that performed HTTP censorship by injecting block pages. 

[Geneva](https://geneva.cs.umd.edu/about/) (*Gen*etic *Eva*sion) is a genetic algorithm we designed to automatically discover new ways to evade censorship, but at its core, Geneva is a packet-level network fuzzer. We modified Geneva's fitness function to reward it for making the elicited response as large as possible, and then trained Geneva against all 184 middleboxes. 

{{< figure src="/weaponizing/packet_sequences.png" >}}

We found 5 packet sequences that elicited amplified responses from middleboxes. Each of these contain a well-formed HTTP GET request for some domain that is forbidden by the middlebox: 

- `SYN` packet (with forbidden request)
- `PSH` packet
- `PSH+ACK` packet
- `SYN` packet, followed by a `PSH` packet containing the forbidden request
- `SYN` packet, followed by a `PSH+ACK` packet containing the forbidden request

We also found another 5 modifications that increased amplification further for a small fraction of middleboxes; an attacker could use these to specific middleboxes. See our paper for more details on these modifications. 

To elicit a response from these middleboxes, we need a domain that is censored or forbidden by each middlebox, but most censoring middleboxes use different blocklists, making it difficult to find one domain that will elicit block pages from everyone. We analyzed the Quack dataset to find the 5 domains that elicited responses from the most middleboxes, which coincidentally spanned five different areas:

- `www.youporn.com` (pornography)
- `www.roxypalace.com` (gambling)
- `plus.google.com` (social media)
- `www.bittorrent.com` (file sharing)
- `www.survive.org.uk` (sexual health/education)

We also used [example.com](http://example.com) and no domain at all as control experiments. 

## Finding Amplifiers

We scanned the entire IPv4 Internet to measure how many IP addresses permit reflected amplification. To do this, we modified the zmap scanner to construct all five packet sequences identified by Geneva. 

We scanned the entire IPv4 Internet a total of 35 times (5 packet sequences × 7 test domains). We measured the responses we got back to calculate the amplification factor we got from each IP address. 

Our version of zmap is open-source and available [here](https://github.com/breakerspace/weaponizing-censors).

# Results

{{< figure src="/weaponizing/amplification-patterns.svg" caption="Types of attacks we find. Thick arrows denote amplification; red ones denote packets that trigger amplification. (a) Normal TCP Reflection, in which the attacker sends a single SYN packet to elicit SYN+ACKs. (b) Middlebox reflection, in which the attacker sends a packet sequence to trigger a block page or censorship response. (c) Combined destination and middlebox reflection, in which the attacker can elicit a response from both the middlebox and end destination. (d) Routing loop reflection, in which trigger packets are trapped in a routing loop. (e) Victim-sustained reflection, in which the victim's default response triggers additional packets from the middlebox or destination. We find that infinite amplification is caused by (d) routing loops that fail to decrement TTLs and (e) victim-sustained reflection.">}}

Recall that we are searching for weaknesses in the TCP implementation in middleboxes, not in the TCP protocol itself. In addition, each middlebox has its own injection policies and block pages: this means that there is no one single amplification factor for this attack, since each middlebox we trigger will be different!

Instead, we can look at the distribution of the response sizes to see the amount of amplification available to attackers. 

Below is a graph of the maximum amplification factor we received for each IP address across all 35 scans, sorted by amplification factor on the x-axis. On the y-axis, you can see the amplification factor that IP address provides. 

On this graph, you can see a huge range in amplification factors - from over 100,000,000 to less than 1. 

{{< figure src="/weaponizing/rank-order-plot.svg" >}}

# Infinite Amplification

Next, we'll examine the IP addresses at the head of the above graph, where we can see amplification factors between 1,000,000 and 100,000,000. These IP addresses are our *mega-amplifiers*, offering tremendous amplification factors. In fact, these amplification factors are likely an *under-estimate;* these numbers are from where our scan stopped collecting data, not when the IP addressed stopped sending us data. What's going on here? 

This is where we find technically *infinite amplification* factors. Amplification factor is calculated by the number of bytes received by an amplifier divided by the number of bytes sent. We found amplifiers that, once triggered by a single packet sequence from the attacker, will send an *endless stream of packets* to the victim. In our testing, some of these packet streams lasted for *days*, often at the full bandwidth the amplifier's link could supply.

We found two causes for this infinite amplification: *routing loops* and *victim sustained amplifiers*. 

### Routing Loops

Routing loops occur between two IP addresses when packets get stuck traversing a loop while being routed from one IP address to the other. Routing loops that contain censoring middleboxes offer a new benefit to attackers: every time the trigger packets circle the routing loop, they re-trigger the censoring middlebox. 

The number of hops a packet will survive in a network is *usually* regulated by the `TTL` field (or 'time-to-live') in IP packets: each time a packet is passed from one router to the next, it's `TTL` value is decremented. If the `TTL` hits 0, the packet is dropped. The maximum `TTL` value is 255: This means an attacker that can send a trigger sequence into an amplifying routing loop get an additional ~250× amplification for free. 

Even more dangerous than normal routing loops are *infinite* routing loops. Infinite routing loops occur if is a circular routing path that does not decrement the `TTL` value, causing packets to circle the loop forever (or, until a random packet drop occurs). We found a small number of infinite routing loops that traversed censorship infrastructure (notably in both China and Russia) that offered infinite amplification. 


{{< figure src="/weaponizing/routingloops.gif" >}}

### Victim Sustained Amplifiers

The second cause we found for infinite amplification was *victim sustained loops.* When a victim receives an unexpected TCP packet, the correct client response is to respond with a `RST` packet. We discovered a small number of amplifiers that will *resend* their block pages when they process *any additional packet* from the victim - including the `RST`. This creates an infinite packet storm: the attacker elicits a single block page to a victim, which causes a `RST` from the victim, which causes a new block page from the amplifier, which causes a `RST` from the victim, etc. 

The victim sustained case is especially dangerous for two reasons. First, the victim's default behavior sustains the attack on itself. Second, this attack causes the victim to flood its own uplink while flooding the downlink. 

{{< figure src="/weaponizing/victimsustained.gif" >}}

# Are these really middleboxes?

Yes, our results suggest so. To test this, we performed a `TTL`-limited experiment: we took the top 1 million amplifiers, tracerouted to them to determine how many hops away they were, and re-sent our probes with a reduced TTL number. This ensures that our probes won't make it to the destination host, but will likely cross the censoring middlebox, so if we still see responses, we know they were generated by a middlebox. We confirmed that ~83% of the top 1 million amplifiers were caused by middleboxes. 

# What is the effect of nation-states?

We found that nation-state censorship infrastructure for countries around the world can also be weaponized. Most of these nation-states are weak amplifiers (the Great Firewall of China only offers about 1.5x amplification, for example), but some of them offer more damaging amplifications, such as Saudi Arabia (~20x amplification). 

The real challenge of nation-states is that their censorship infrastructure usually processes all traffic entering or exiting the country. This means that unlike other amplification attacks, where the source IP address of the traffic received by the victim is the amplifier itself, every IP address behind a middlebox can appear as the source of traffic. Said another way: *every IP address* within an amplifying nation-state can be an amplifier. 

# Attack Damage

How much damage can an attacker do with infinite amplification? This is a case in which the 'amplification factor' as a metric starts to break down. Most of the time, when people ask about amplification factor, they are asking how much damage an attacker can do with a given attack vector. If an attacker can elicit technically infinite amplification factor, but at only 64 kbps before the link is completely saturated, the amount of damage an attacker can do is limited.

A better question: what is the maximum bandwidth an attacker can elicit through this attack? Unfortunately, this is the hardest to study ethically. To measure the maximum capacity of a given amplifier, we would have to completely saturate the link for each network, which could have real negative consequences on the users of that network. For now, the true capacity available to attackers from this attack is unknown.

# Defenses

Defending against this attack is difficult. The incoming flood of traffic come over TCP port 80 (normal HTTP traffic) and the responses are usually well-formed HTTP responses. 

Since middleboxes are spoofing the IP address of the traffic they generate, this means that the attacker can set the source IP address of the reflected traffic to be *any IP address* behind the middlebox. For some networks, this is a small number of IP addresses, but if an attacker uses nation-state censorship infrastructure, the attacker can make the attack traffic come from any IP address within that country. This makes it difficult for a victim to drop traffic from offending IP addresses during an attack. 

# Responsible Disclosure

In September of 2020, we reached out and shared an advanced copy of our paper with several country-level CERTs (Computer Emergency Readiness Teams), DDoS mitigation services, and firewall manufacturers. We also had further meetings with multiple DDoS mitigation services and US CERT for further discussions about mitigations, and have been in ongoing communication with DDoS mitigation services.

Unfortunately, there is only so much we can do. Completely fixing this problem will require countries to invest money in changes that could weaken their censorship infrastructure, something we believe is unlikely to occur. 

# FAQ

1. **Who is vulnerable? Do I need to live within a censored country to be a victim?**  
Unfortunately, we found that this attack can directed at practically anyone, whether you live inside a censored regime or not. 
2. **Who is vulnerable to being weaponized?**  
Most nation-state censorship infrastructure is currently vulnerable, as well as many off-the-shelf commercial firewalls. Our study was tailored to provide a global view of this issue, not at individual firewall types, so we do not yet know the full list of firewalls in active use that are vulnerable to this.
3. **How big of an amplification factor can attackers get with this attack?**  
  We are taking advantage of the *implementation* of the TCP protocol by middleboxes, unlike most prior attacks which take advantage of a protocol specification itself. As a consequence, we can't specify just one number for our amplification factor: it's different from every single middlebox!  
  &nbsp;  
  In total though, we found hundreds of thousands of IP addresses that exceeded amplification factors from DNS and NTP, and hundreds of that offered amplification factors greater than the famed memcached (51,000x). 

4. **What was the highest amplification factor you discovered?**  
  Technically, *infinite.* Amplification factor is calculated by the number of bytes received by an amplifier divided by the number of bytes sent. We found amplifiers that, once triggered by a single packet sequence from the attacker, will send an endless stream of packets to the victim. In our testing, some of these packet streams lasted for *days*, and some at the full bandwidth their link could supply.  
  &nbsp;  
  This is a case in which the 'amplification factor' as a metric starts to break down, however. Most of the time, when people ask about amplification factor, they are asking how much damage an attacker can do with a given attack vector. If an attacker can elicit an infinite amplification factor, but at only at a maximum speed of 64 kbps, the amount of damage an attacker can do is limited.  
  &nbsp;  
  A better question: what is the maximum bandwidth an attacker can elicit through this attack? Unfortunately, this is the hardest to study ethically. To measure the maximum capacity of a given amplifier, we would have to completely saturate the link for each network, which could have real negative consequences on the users of that network. For now, the true capacity available to attackers from this attack is unknown.

5. **Who needs to fix this?**  
  There are two angles to fix this issue: preventing attackers from spoofing their source IP address and protecting middleboxes from incorrectly injecting traffic. Unfortunately, completely preventing source IP address spoofing has been an ongoing effort for years. Fixing this issue completely will require every vulnerable firewall manufacturer to update their middleboxes and every organization and censoring nation state that has deployed a susceptible middlebox to upgrade their infrastructure.  
  &nbsp;  
  This issue is unfortunately unlikely to be resolved completely anytime soon. 
6. **Should I be worried?**  
Most individuals likely do not need to worry about this attack, but we've been working with DDoS mitigation services so they are prepared if the attack is used in the wild. 
7. **Has this attack been detected in the wild?**   
Not yet, to our knowledge. We're working with different DDoS mitigation services to monitor if it is used. 
8. **How does this compare to previous amplification attacks?**  
Our results suggest that this attack is at least as dangerous as the largest existing UDP-based amplification attacks. We found hundreds of IP addresses with middleboxes that offered amplification factor larger than memcached (51,000x), and hundreds of thousands of IP addresses that offered amplification factors greater than DNS and NTP. 
9. **Was this attack responsibly disclosed?**  
Yes, but we're limited. In September of 2020, we reached out and shared an advanced copy of our paper with several country-level CERTs, DDoS mitigation services, and firewall manufacturers. We also met with multiple DDoS mitigation services and US CERT for further discussions about mitigations.  
  &nbsp;  
  Unfortunately, completely fixing this problem will require countries to invest money in changes that could weaken their censorship infrastructure, which we do not think is likely to occur. 
10. **Why don't you have a logo or fancy name for your attack?**  
Don't tempt us. 

# Paper

You can read our full paper [here](https://geneva.cs.umd.edu/papers/usenix-weaponizing-ddos.pdf). To cite, please use the Bibtex [here](https://www.usenix.org/biblio/export/bibtex/272318). 

