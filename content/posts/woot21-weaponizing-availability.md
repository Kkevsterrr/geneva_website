---
title: "Weaponizing Censorship Infrastructure for Availability Attacks"
shorttitle: "Weaponizing Availability"
featured_image: "/fiber.jpg"
show_particles: false
date: 2021-08-11T03:53:38-05:00
description: "Attackers can force censors to block arbitrary IP address pairs."
summary: "We discover it is possible for an attacker to weaponize a censor to prevent _any pair_ of hosts from communicating across its borders by abusing a little known feature of many censorship systems: _residual censorship_."
draft: false
author: "Kevin Bock, Pranav Bharadwaj, Jasraj Singh, and Dave Levin"
---

This work appeared in WOOT 21 "[Your Censor is My Censor: Weaponizing Censorship Infrastructure for Availability Attacks](/papers/woot21-weaponizing-availability.pdf)". You can also view the [conference talk](https://www.youtube.com/watch?v=QkB3IeHpGgY). 

**Summary:**

- We discover a new way that attackers could weaponize censorship infrastructure to cause  *a* denial of service attacks between any pair of IP addresses that cross the censor.
- The attack is completely off-path and exploits a little known feature of many censorship systems called *residual censorship.*
- This attack allows effectively anyone to weaponize censorship infrastructure to perform their own targeted blocking.

**Collectively, our results show that censorship infrastructure poses a greater threat to the broader Internet than previously understood.** 

See our full paper [here](https://geneva.cs.umd.edu/papers/woot21-weaponizing-availability.pdf), our open-source and artifact evaluated code [here](https://github.com/breakerspace/weaponizing-residual-censorship/), our [talk](https://www.youtube.com/watch?v=QkB3IeHpGgY), or you can jump to the [FAQ section](#faq) below. 

{{% callout emoji="" color="blue" %}}

This is not the only way we've discovered that censors could be weaponized: see the post about our work in USENIX Security 2021 "[Weaponizing Middleboxes for TCP Reflected Amplification](/weaponizing/)".

{{% /callout %}}

# What is residual censorship?

Residual censorship is a feature observed in some censorship systems in which the censor
continues to block innocuous requests for a short period of time (usually two minutes or less)  *after* censoring a forbidden request. 

An important facet of residual censorship is what information the censor remembers about the client's connection after censorship is initially triggered. There are three basic options available to an adversary: *2-tuple* (in which the censor remembers the client's IP address and. the server IP address), *3-tuple* (client IP address, server IP address and port), or *4-tuple* (client IP address+port, server IP address+port). The less specific information the censor remembers, the broader and harsher its residual censorship is. 

{{< figure src="/weaponizing/types_of_residual.001.png" >}}

At this time, we aren't aware of any censorship systems that employ 2-tuple residual censorship, so our work focuses on 3-tuple and 4-tuple residual censorship. 

# Weaponizing residual censorship

To weaponize residual censorship, our idea is straightforward. The attacker will spoof the source IP address of the victim client and generate a forbidden request to the victim server. This will activate residual censorship and block the client and server from communicating for a short period of time. 

{{< figure src="/weaponizing/weaponizing_residual.gif" >}}

We found that the *direction* of residual censorship matters: this attack will prevent the client from communicating with the server, but will not always prevent the server from communicating with the client. 

How can an attacker generate a forbidden request for TCP based protocols while spoofing their source IP address? In prior work, we used Geneva to identify packet sequences that could be used to trigger censors without requiring a proper 3-way handshake, even for stateful censors that try to monitor the 3-way handshake.

# Ethically demonstrating the attack

Since all of our vantage points employed egress filtering, we cannot launch the attack directly from our censored vantage points within China, Iran, or Kazakhstan. 

Instead, we leverage a public deployment of [SP3](https://github.com/willscott/sp3) (A Simple Practical & Safe Packet Spoofing Protocol) deployed at the University of Washington, to ethically send source-spoofed packets and thus act as our attacker. SP3 is a web server that offers the ability to send spoofed packets, but mandates that a client *consent* to receiving source-spoofed packets. A client gives this consent by creating and holding open a websocket connection to SP3. When the client connects, SP3 returns a UUID16 challenge string. As long as the websocket connection is held open, other servers can connect to SP3 with a websocket, supply the challenge code, and can give SP3
packets through binary frames to send to that client.

We launched the attack on ourselves as follows. We used SP3 to send a sequence of packets to  trigger residual censorship to a server that crosses the censor, with the source addresses spoofed to be a test victim under our control. Recall that traffic direction matters to residual censorship: the attacker must be on the same side of the censor as the victim. Since SP3
is located in the United States, this means we are launching the attack from outside-in for each censoring country. Our vantage points within each country acted as the server; we launched the attack against all of our geographically disparate vantage points around the world as
victims.

Then, we used our “victim” to make requests to the server, and recorded if the connection succeeded or if it was impacted by residual censorship. We varied our test request based on the protocol and type of residual censorship. For 3-tuple residual censorship, the client makes an innocuous request with a different source port to the same server IP address and port. For 4-tuple residual censorship, we ensure the client uses the same source port as the attacker. Of course, in a real attack scenario, the attacker cannot know the source port a victim will use a priori. Therefore, to weaponize 4-tuple residual censorship systems, the attacker would re-trigger censorship for all 65,535 possible source ports. As we'll see later, this does not impose a significant limitation on attackers. 

{{< figure src="/weaponizing/exp.gif" >}}

### Results

We launched this attack against our uncensored vantage points located around the world in Australia (Sydney), India (Mumbai), Ireland (Dublin), Japan (Tokyo), United Arab Emirates (Dubai), and the United States (Iowa, Colorado, and Virginia) for every bidirectional, residually censored protocol to each of our vantage points in China, with HTTP and ESNI, in Kazakhstan, with HTTP and SNI, and in Iran, with HTTP and SNI. 

In every country we tested, **we could successfully weaponize the censorship infrastructure against every victim vantage point at least once around the world.** We find that the attack is sensitive to the chosen protocol (for example, HTTPS offers better results in Kazakhstan than HTTP).

Collectively, our results suggest that there are many shared paths through the censorship infrastructure of each country, and an attacker that can access just one source spoofed capable
machine is capable of launching highly effective availability attacks. A more well resourced attacker could likely get even better results by choosing vantage points with even more similar paths as their victims.

See our paper for the full breakdown of success rates by attack location. 

# Sustaining the attack

Since residual censorship usually only lasts for a short period of time, how can an attacker sustain the attack? It depends on the type of residual censorship. 

For 4-tuple residual censorship, it is a little harder, since the attacker can't guess the client's source port. Therefore, the attacker has to send the trigger packet sequence 65,535 times (once for each source port) within the duration of residual censorship. In Kazakhstan, this requires sending 145 bytes 65,535 times within 120 seconds, requiring 634 kbps to sustain the attack indefinitely. 

For 3-tuple residual censorship, the attacker must be able to send the trigger sequence at least once per duration of residual censorship. In China, this requires the attacker to send 145 bytes every 90 seconds: requiring only 13bps. A weak attacker can sustain this attack easily.

In some cases though, we discovered that the victim themselves help the attacker sustain the attack. In Kazakhstan and Iran, any additional packets sent by the victim reset the censor's residual censorship timer. This means the victim's *default behavior* (resending unacknowledged packets) prolongs the attack all on its own. 

# Code

All of our code is open source and available on Github [here](https://github.com/breakerspace/weaponizing-residual-censorship/). Our code completed Artifact Evaluation with the conference and was awarded the Results Reproduced (ROR-R) and Open Research Artifacts (ORO) badges. 

By releasing this code, are we helping attackers? We don't think so. The code we are releasing is useful for evaluating this attack, but not useful for deploying the attack in the wild. This is because our code goes through SP^3: an attacker can only use our code to launch an attack if they are attacking themselves. 

# Paper

In our paper, we perform an in-depth look at the current state of residual censorship in Iran, Kazakhstan, and China: the types of residual censorship in operation, their censorship mechanisms, and their limitations. There are also more details in our paper about our experiment configuration. See our full paper [here](https://geneva.cs.umd.edu/papers/woot21-weaponizing-availability.pdf) for more details. 

# FAQ

1. **Who is vulnerable? Do I need to live within a censored country to be a victim?**  
This attack blocks any pair of IP addresses from communicating across a censor. This means that you could be affected, but it's less likely. If you live outside a censored regime, an attacker could prevent you from accessing a resource inside that censored regime. 
2. **Who is vulnerable to being weaponized?**  
We studied Iran, Kazakhstan, and China in our paper and demonstrated this attack live against all three countries. 
3. **What can a client or server do to defend themselves?**  
Unfortunately, there isn't much a client or server can do if this attack is used. This attack effectively coopts the strengths of the national censorship infrastructure, and we were unable to find any way for a client or server to disable the attack or defend themselves.
4. **What can we do to detect this attack?**   
Servers could try to monitor for packet sequences used by attackers to trigger residual censorship, but this approach is not foolproof. Since attackers only need the censor to process their packet sequence, an attacker can abuse the `ttl` (time to live) field to hide the attack. The `ttl` field determines how long packets can survive in a network: it decrements once per hop, and if the `ttl` field hits 0, the packet is dropped. An attacker can set the `ttl` of their packets high enough to cross the censor, but not high enough to reach the server. In these cases, the attack would be all but invisible. 
5. **Has this attack been detected in the wild?**   
Not yet, to our knowledge, but we believe detecting it would be extremely difficult. 
6. **Is your code open source?**  
Yes: our code is open source and available on Github [here](https://github.com/breakerspace/weaponizing-residual-censorship/). Our code completed Artifact Evaluation with the conference and was awarded the Results Reproduced (ROR-R) and Open Research Artifacts (ORO) badges. 
7. **By releasing this code, aren't you helping attackers?**  
There's always a small risk of this, but in this case, we don't think so. The code we are releasing is useful for *evaluating* this attack, but not useful for *deploying* the attack in the wild. This is because our code relies entirely on SP^3, which means an attacker can only use our code to launch an attack if they are attacking themselves. 
8. **Was this attack responsibly disclosed?**  
Yes. We reached out to several country-level CERTs about this issue to report it, but never heard back. Unfortunately, completely fixing this problem will require countries to invest money in potentially costly changes that could weaken their censorship infrastructure, which we do not think is likely to occur. 


