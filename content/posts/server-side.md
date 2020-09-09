+++
title =  "Evading Censorship from the Server-side"
lang = "en"
date = 2020-09-08
tag = []
featured_image = "server-side-cover.jpg"
summary = "Using our tool Geneva, we have discovered how to circumvent censorship from the server-side: with no client participation whatsoever. This opens up new avenues for helping people evade censorship, even if they didn't realize they were being censored in the first place."
+++

# Evading Censorship from the Server-side

**Summary:  Using our tool Geneva, we have discovered how to circumvent censorship from the server-side: *with no client participation whatsoever*. This opens up new avenues for helping people evade censorship, even if they didn't realize they were being censored in the first place.**

**This is the first in a series of blog posts on [our most recent paper on server-side evasion](https://geneva.cs.umd.edu/papers/come-as-you-are.pdf) where we will be covering: what it means to evade censorship from the server-side, specific server-side evasion strategies, and what new things it allows us to learn about censors.** 

---

All existing censorship evasion tools and techniques have involved some involvement from the client. This seems intuitive: to evade censorship, shouldn't the client have to do *something*? Regardless of the approach, be it Tor, proxies, VPNs, protocol obfuscation, or even [Geneva's previous strategies](https://geneva.cs.umd.edu/papers/geneva_ccs19.pdf) (to name a few), the client has always been required to install some extra software.

Unfortunately, active participation on the part of clients can limit the reach of censorship evasion techniques. In some scenarios, installing anti-censorship software can put [users at risk](https://www.nytimes.com/2009/05/01/technology/01filter.html). For users who are willing to take on this risk, it can be difficult to bootstrap censorship evasion, as the anti-censorship tools themselves [may be censored](https://blog.torproject.org/internet-censorship-iran-findings-2014-2017). Worse yet, there are many users who do not seek out tools to evade censorship because they [do not even know they are being censored](https://www.nytimes.com/2018/08/06/technology/china-generation-blocked-internet.html).

# Server-side evasion can help

Consider what *server-side* evasion would look like: suppose a server *outside* a censoring regime could alter its traffic in a way that lets censored clients access content without having to install any extra software. Server-side evasion would allow servers located outside of a censoring regime to subvert censorship *on behalf of clients.* 

In this work, we specifically examine server-side evasion techniques against DPI-based censorship in China, India, Iran and Kazakhstan. With DPI-based censorship this model, clients and servers can communicate, but the censor will step in and teardown the connection if forbidden content is exchanged. Many other countries use DPI, as well, and we expect our approach to generalize. However, these techniques do not necessarily extend to a censor directly blocking IP addresses. 

# ... but it shouldn't be possible

Unfortunately, at first glance, defeating censorship purely from the server-side seems impossible. The server would have to somehow subvert the censor before the client sends the forbidden request. The challenge is that the server does almost *nothing* before then. To see this, consider the packets that are exchanged leading up to a forbidden HTTP `GET` request. 

{{< figure src="/server-side-handshake.png" caption="A censored HTTP GET request sends the forbidden keyword immediately following the TCP three-way handshake.">}}

First, the client would initiate a TCP three-way handshake, during which the client sends a `SYN`, the server responds with a `SYN+ACK`, and the client responds with an `ACK`. Then, the client would send a `PSH+ACK` packet containing the HTTP request with the censored keyword, at which point the censor would tear down the connection (e.g., by injecting `RST` packets to both the client and the server). The *only* packet a server sends before a typical censorship event is just a `SYN+ACK`—this would seem to leave very little room for a censorship evasion strategy.

It is tempting to assume we can re-use client-side strategies from prior work (such as [those found by Geneva](https://geneva.cs.umd.edu/papers/geneva_ccs19.pdf)). Unfortunately, we find that client-side strategies do not generalize to server-side. Packets from clients and servers are treated differently, and shortcomings in censors identified in the past from the client-side do not necessarily lend insight to server-side evasion. (See [our paper](https://geneva.cs.umd.edu/papers/come-as-you-are.pdf) for more details on how we tested this.) Evading from the server-side required a blank-slate approach.

# Training Geneva

{{% callout emoji="" color="blue" %}}
**Geneva** (**Gen**etic **Eva**sion) is a genetic algorithm we developed that discovers censorship evasion strategies against a censor. Unlike most anti-censorship systems, it does not require deployment at both ends of the connection: it runs exclusively at one side (client or server) and defeats censorship by manipulating the packet stream to confuse the censor without impacting the underlying connection. A censorship evasion strategy describes how that traffic should be modified. Since Geneva will be evolving these strategies, they are expressed in a domain-specific language. (For a full rundown of Geneva’s strategy syntax, see [our GitHub page](https://github.com/Kkevsterrr/geneva) or [our documentation](http://geneva.readthedocs.io/)).

Geneva is comprised of two main components. First, the genetic algorithm, which can evolve new ways to defeat a censorship system given an application that experiences censorship and a fitness function. Second, the strategy engine, which applies a given strategy to modify active network traffic.
{{% /callout %}}

We modified Geneva so it could run from the server-side and added plugins to train with DNS-over-TCP, FTP, HTTPS, and SMTP. 

Over the span of five months, we ran Geneva server-side in six countries—Australia, Germany, Ireland, Japan, South Korea, and the US—on five protocols: DNS (over TCP), FTP, HTTP, HTTPS, and SMTP (all over IPv4). We used unmodified clients within four nation state censors—China, India, Iran, and Kazakhstan—to connect to our servers. For each nation-state censor, we trained on each protocol for which we were able to trigger censorship; all four countries censored HTTP, but only China censored all five protocols. 

# Server side evasion is possible!

In total, we found **11 strategies** in [our original study](https://geneva.cs.umd.edu/papers/come-as-you-are.pdf) (and another 4 server-side strategies against [China's new ESNI censorship system](https://geneva.cs.umd.edu/posts/china-censors-esni/esni)) spanning China, India, Iran, and Kazakhstan across every protocol they censor (HTTPS, HTTP, DNS, FTP, SMTP).  

To see how server-side evasion can work, let's examine one of the strategies from our paper that successfully circumvents censorship in Kazakhstan: the `Double Benign GET`.

{{< figure src="/server-side-benign-get.png" caption="A server-side evasion strategy that is successful in Kazakhstan. The server's strange data-bearing SYN/ACKs confuse the censor." >}} 

During the three-way handshake, instead of sending one `SYN/ACK` packet like usual, the server sends *two* `SYN/ACK` packets. On each packet, it attaches a payload: a well-formed HTTP GET request for some benign resource (`GET / HTTP/1.1\r\nHost: example.com\r\n\r\n`). This evaded Kazakhstan's censorship with 100% success rate in our tests.

Why does this work? As we understand it, the two `GET` requests confuse the censor as to who is the client and who is the server. The `GET` requests lead the censor to believe the server is actually the client, and because the Kazakhstani censor ignores requests from (who it believes to be) the server, it subsequently ignores the *actual* client's real request. For the rest of this connection, the client can freely communicate through the censor.

If this confuses censors, does it also confuse clients? We tested on many flavors of Windows, Mac, Linux, and mobile devices, and all of the strategies worked for all clients, with a small number of exceptions for Windows: We found only one type of strategy that did not work on Windows, but after slight modification, we can support Windows end hosts. There appears to be a large space of strategies that exploit bugs and errors in middleboxes that don't exist in the endhosts, and by training on real machines against real censors, Geneva can find them.

# Up next

In this post—the first in a series—we have shown that server-side censorship evasion is possible without any extra client-side software whatsoever. In the next posts, we'll delve deeper into some of the server-side evasion strategies we discovered in China and what they teach us about how censorship operates.

# Try it yourself

Geneva is open-source [on our GitHub page](https://github.com/Kkevsterrr/geneva). If you want to try these strategies out yourself, it is simple to do so. Using the strategy engine (`engine.py`), you can run the strategies in front of an existing server to make it more resilient to censorship. Follow the instructions on [our Github page](https://github.com/kkevsterrr/geneva).


{{% callout emoji="" color="red" %}}
Disclaimer: Geneva's strategies intentionally take overt action on the network to interfere with the normal operation of censors. The genetic algorithm in Geneva trains against a live adversary, so when running the genetic algorithm to evolve new censorship strategies, it will intentionally trigger censorship many times during training. Understand the risks of using Geneva in your country before trying it.
{{% /callout %}}

