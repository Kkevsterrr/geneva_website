+++
title =  "Exposing and Circumventing China's Censorship of ESNI"
lang = "en"
date = 2020-08-07
tag = []
featured_image = "red_locks.jpeg"
summary = "The Great Firewall (GFW) of China has recently begun blocking ESNI—one of the foundational features of TLS 1.3 and HTTPS. We empirically demonstrate what triggers this censorship and how long residual censorship lasts. We also present several evasion strategies discovered by Geneva that can be run either client-side or server-side to evade blocking."
authors = "Kevin Bock, iyouport, Anonymous, Louis-Henri Merino, David Fifield, Amir Houmansadr, Dave Levin"
+++

On 2020-07-30, [iyouport](https://www.iyouport.org/) [reported](https://mailarchive.ietf.org/arch/msg/tls/YzT5LjLJ_6WWhdnU2wVsKNKR6_I/) ([archive](https://web.archive.org/web/20200801221253/https://mailarchive.ietf.org/arch/msg/tls/YzT5LjLJ_6WWhdnU2wVsKNKR6_I/)) the apparent blocking of TLS connections with the encrypted SNI (ESNI) extension in China.
iyouport says that the first occurrence of blocking was one day earlier, on 2020-07-29.

We confirm that the Great Firewall (GFW) of China has recently begun blocking ESNI—one
of the foundational features of TLS 1.3 and HTTPS.  We empirically demonstrate
what triggers this censorship and how long residual censorship lasts.  We also
present several evasion strategies discovered by
[Geneva](https://geneva.cs.umd.edu) that can be run either client-side or
server-side to evade blocking.

## What is Encrypted Server Name Indication (ESNI)?

TLS is the foundation of secure communication on the web (HTTPS). It provides
authenticated encryption so that users can know with whom they are
communicating, and that their information cannot be read or tampered with by an
intermediary.  Although TLS hides the *content* of a user's communication, it
does not always hide *with whom* the user is communicating; the TLS handshake
optionally contains a Server Name Indication (SNI) field that allows the user's
client to inform the server which website it wishes to communicate with.
Nation-state censors have used the SNI field to block users from being able to
communicate with certain destinations.  China, for one, has long been censoring
HTTPS in this manner.

TLS 1.3 introduced Encrypted SNI (ESNI) that, put simply, encrypts the SNI so
that intermediaries cannot view it.  (To learn more about ESNI and its
benefits, see [Cloudflare's
article](https://blog.cloudflare.com/encrypted-sni/)).  ESNI has [the potential
to complicate nation-states' abilities to censor HTTPS content](https://www.usenix.org/system/files/foci19-paper_chai_update.pdf); rather than be
able to block only connections to specific websites, ESNI would require censors
to block all TLS connections to specific servers.  We do confirm that this is now
happening in China!

## Our Main Findings

* The GFW blocks ESNI connections by dropping packets from client to server.
* The blocking can be triggered bidirectionally.
* The `0xffce` extension is necessary to trigger the blocking.
* The blocking can happen on all ports from 1 to 65535.
* Once the GFW blocks a connection, it will continue blocking all traffic associated with the 3-tuples of (srcIP, dstIP, dstPort) for 120 or 180 seconds.
* We have discovered 6 client-side and 4 server-side evasion strategies.

## How Do We Know These?

We have made a simple Python program that performs the following:

1. completes a TCP handshake with a specified server;
2. and then sends a TLS ClientHello message with an ESNI extension; the fingerprint of the ClientHello is as normal as what Firefox 79.0 would send.

The program sends ClientHellos with ESNI both inside-out and outside-in, while
capturing traffic on both sides for analysis. The servers to which we send
ClientHellos complete the TCP handshake, but they do not send any data packets
back to the client, nor do  they are first to close the connection.  All
experiments were conducted between July 30th and August 6st.

## Details About the Blocking

### Blocking by dropping packets, not injecting RSTs

Comparing the traffic captured on both endpoints,
we find the GFW blocks ESNI connections by dropping packets from clients to servers.

This has two differences from how the GFW censors other commonly-used
protocols.  First, the GFW censors (non-encrypted) SNI and HTTP by injecting
forged TCP RSTs to both server and client; conversely, we have observed no
injected packets from the GFW to censor ESNI traffic.  Second, the GFW drops
traffic from server to client to block Tor and [Shadowsocks](https://gfw.report/blog/gfw_shadowsocks) servers; however, it
drops only client-to-server packets when censoring ESNI.

We further note the GFW does not distinguish the flags of TCP packets when
dropping them.  (This is different from some censorship systems in Iran which do
not drop packets with RST or FIN flags.)

### The blocking can be triggered bidirectionally

We find the blocking can be triggered bidirectionally.  In other words, sending
an ESNI handshake from outside the firewall to inside can get blocked in the
same way as sending it inside-out.

Thanks to this bidirectional feature, one can test this ESNI-based censorship
remotely from the outside of the GFW without having control of any Chinese
server.  The GFW's censorship on DNS, HTTP, SNI, FTP, SMTP, and Shadowsocks can
also be measured outside-in.

#### The GFW censors ESNI, but not omit-SNI

We confirm a TLS ClientHello without ESNI/SNI extensions cannot trigger the
blocking.  In other words, the `0xffce` payload of the `encrypted_server_name`
extension is necessary to trigger the blocking.

We tested this by replacing the `0xffce` in a triggering ClientHello with `0x7777`.
After the replacement, sending such a ClientHello could not trigger the blocking
anymore.

This confirmation is important because some censors have been observed blocking [any ClientHello message without the SNI extension](https://github.com/net4people/bbs/issues/10#issuecomment-532035677),
which would result in the blocking of both ESNI and [omitting-SNI](https://tlsfingerprint.io/static/frolov2019.pdf).

#### New extension values are not blocked

As informed by an anonymous reviewer on the [riseup pad](https://pad.riseup.net/p/xCRfphD5CoxmbFcpc1s2),
the currently deployed ESNI uses extension value `0xffce` (see [Section 8.1](https://datatracker.ietf.org/doc/html/draft-ietf-tls-esni-01)).
However, the newer ECH uses extension value `0xff02`, `0xff03` and `0xff04`([Section 11.1](https://datatracker.ietf.org/doc/html/draft-ietf-tls-esni-07)).
We confirm no censorship has been observed on these extension values yet.

Specifically,
we replace the `0xffce` in a triggering ClientHello with the values of `0xff02`, `0xff03`, and `0xff04` respectively.
And no blocking is observed after sending such modified ClientHellos.

#### A complete TCP handshake is required before triggering the blocking

We find a complete TCP handshake is necessary in order to trigger the ESNI blocking.

We conducted two experiments from the outside to a server in China.
In the first experiment,
without sending any `SYN` packet,
our client sent one naked ClientHello message with ESNI extension every 2 seconds.
In the second experiment,
our client sent a `SYN` packet and a ClientHello message with ESNI extension;
but the server would not respond with any packet (not even to complete the TCP three-way handshake).

In total, we sent 10 ClientHello messages in each experiment.
The result shows no blocking or residual censorship was ever triggered; all ClientHello messages reached the server.
This means a TCP handshake is necessary before triggering ESNI-based censorship.
It also indicates, similar to the SNI-based censorship by the GFW, the censorship machine for ESNI is stateful.

#### The blocking can happen on all ports

We find the ESNI blocking can happen not only on port 443,
but on *all* ports from 1 to 65535.

Specifically, we sent two ESNI handshakes in a row to the port 1-65535 of a Chinese server from the outside.  For each port, we first sent an
ESNI handshake; then after the connection timeout (after 20 seconds), we tried
to complete a TCP handshake with the server again. If we do not receive any
`SYN+ACK` from the server the second time, we consider the censorship occurred on
that port.  As a result, the ESNI blocking was observed on all ports from 1 to
65535.

This feature allows us to test ESNI censorship efficiently, as we can conduct
testings on multiple ports of the same IP address simultaneously.

### Residual Censorship

We find that the GFW employs "residual censorship" of ESNI connections. This means
that, for some amount of time after triggering censorship for a given connection,
it will continue blocking *any* connections with the same 3-tuple of source IP, destination IP, and destination port.

The precise duration of residual censorship appears to vary by vantage point.
We observed residual censorship for 120 seconds at two of our vantage points,
and 180 seconds at another vantage point.

Sending additional ESNI handshakes during residual censorship time does *not* reset the timer of the censoring machine.
This is similar to the previously observed residual censorship on SNI-based blocking of the GFW.
(Conversely, each additional packet set while residual censorship in effect in
[Iran resets the timer](https://geneva.cs.umd.edu/posts/iran-whitelister/).)

These findings are partially based on the following experiment.
From the outside, we sent one ClientHello message per second to port 443 of a
Chinese server.  The 1st, 2nd, and 121st TCP handshakes were accepted.
All other handshake attempts were unsuccessful because the `SYN`s did not
reach the server.

This result shows, similar to previously discovered SNI-based residual censorship,
the GFW also employs residual censorship for ESNI.
In addition, the fact that second handshake could complete means that it takes at least 1 second for the GFW to react and enable the blocking rules.

## How Can We Circumvent the Blocking?

<!-- callout section open -->

**Geneva (*Gen*etic *Eva*sion)** is a genetic algorithm developed by those of
us at the University of Maryland that automatically discovers new censorship
evasion strategies.  Geneva manipulates packet streams—injecting, altering,
fragmenting, and dropping packets—in a manner that bypasses censorship without
impacting the original underlying connection.  Unlike most other anti-censorship
systems, Geneva does not require deployment at both sides of the connection:
it runs exclusively at one side (client or server).

Geneva trains its genetic algorithm against live censors, and to date has found
dozens of censorship evasion strategies in various countries.  Geneva's
strategies are expressed in a domain-specific language.  Details of the
language, along with the entire Geneva codebase, are available at the [Geneva
GitHub repository](https://github.com/kkevsterrr/geneva).

To learn more about how Geneva (or the Geneva strategy engine) works under the
hood, see
our [papers](https://geneva.cs.umd.edu/papers) or [about](https://geneva.cs.umd.edu/about) page.


<!-- callout section close -->

To allow Geneva to train directly against the GFW's ESNI censorship, we wrote
a custom plugin that performs the following steps:

1. Geneva starts a TCP server on a random open port on a vantage point located outside of China. By randomizing our ports, we do not need to worry about residual censorship.
2. Geneva drives a TCP client located inside of China to connect to the server.
3. The client sends a TLS 1.3 ClientHello with the Encrypted SNI extension.
4. The client sleeps for 2 seconds to allow the GFW censorship to kick in.
5. The client sends a short test message `"test"` to test if it has been censored.
6. Steps 4 & 5 are repeated.
7. The server confirms that it receives both the full TLS ClientHello from the client and the test messages. If it does, the strategy is rewarded with a positive fitness; if not (or if the client timed out while sending its test messages), the strategy is punished.

With this, Geneva discovered multiple evasion strategies *in just a few hours*.
We describe them in detail below.

The Geneva strategy engine is open source on [our
Github](http://github.com/kkevsterrr/geneva).

All of these strategies can be run with our open-source Geneva strategy engine ([repository](http://github.com/kkevsterrr/geneva)).  Since they operate at the TCP layer, they can be
applied to any application that needs to use ESNI: with Geneva running, even an
unmodified web browser can become a simple censorship evasion tool.


Note that Geneva is *not* designed as a general purpose evasion tool, and does
not provide any additional encryption, privacy, or protection. It is a research
prototype and it is not optimized for speed. Use these strategies at your own
risk.

### Evasion strategies

We trained Geneva over the span of 48 hours, both client- and server-side. In
total, we discovered 6 strategies to defeat the ESNI censorship: 4 that work
from the server, and 6 that work from the client.

The following are TCP-layer strategies that can defeat the ESNI censorship when applied exclusively at the client-side.

**Strategy 1: Triple `SYN`**

The first client strategy works by initiating the TCP 3-way handshake with
*three* `SYN` packets, such that the sequence number of the third `SYN` is
corrupted.

In Geneva's syntax, this strategy looks like this: `[TCP:flags:S]-duplicate(duplicate,tamper{TCP:seq:corrupt})-| \/`

This strategy performs a desynchronization attack against the Great Firewall.
The GFW synchronizes on the corrupt sequence number, so it misses the ESNI
request.

This strategy can also be applied from the server-side:

`[TCP:flags:SA]-tamper{TCP:flags:replace:S}(duplicate(duplicate,tamper{TCP:seq:corrupt}),)-| \/`

Although this strategy makes it so the server never sends a `SYN+ACK` packet,
this does not break the three-way handshake. During the three-way handshake,
instead of the server sending a `SYN+ACK` packet as usual, the server instead
sends three `SYN` packets (the third with a corrupt sequence number).

The first `SYN` packet serves to initiate a TCP Simultaneous Open, an archaic
feature of TCP supported by all major operating systems to handle the case in
which two TCP stacks send a `SYN` packet at the same time. When the client
receives a `SYN` from the server, the _client_ sends a `SYN+ACK` packet, and
server responds with an `ACK` to complete the handshake. This effectively
changes the traditional three-way handshake to a four-way handshake. The `SYN`
with the corrupt sequence number causes the GFW to desynchronize (but is ignored
by the client), successfully defeating censorship without harming the
connection.

**Strategy 2: Four Byte Segmentation**

The next strategy we discover can also be used from client or server. In this
strategy, the client sends the ESNI request across two TCP segments, such that
the first TCP segment is less than or equal to 4 bytes long.

From the client-side, in Geneva's syntax this strategy looks like this: `[TCP:flags:PA]-fragment{tcp:4:True}-| \/`

This is not the first time Geneva has discovered segmentation strategies, but
it is surprising that this strategy works in China. The Great Firewall has
been famous for its ability to reassemble TCP segments for almost a decade now (see
[brdgrd](https://github.com/NullHypothesis/brdgrd)). The TLS header is 5 bytes
long, so by segmenting specifically the TLS header across multiple packets, we hypothesize
this breaks the GFW's ability to protocol fingerprint ESNI packet as TLS. This
has interesting implications for how the GFW fingerprints connections: it
suggests the component of the GFW that performs connection fingerprinting
cannot reassemble TCP segments for all protocols. This theory is supported by
other segmentation-based strategies identified by Geneva in the past (see [this
paper](https://geneva.cs.umd.edu/papers/come-as-you-are.pdf)).

This strategy can also be triggered from the server-side. By reducing the TCP
window size during the 3-way handshake, a server can force the client to segment
their request. In Geneva's syntax, this can be accomplished with:
`[TCP:flags:SA]-tamper{TCP:window:replace:4}-| \/`.

**Strategy 3: TCB Teardown**

The next strategy is a classic TCB (TCP Control Block) Teardown: the client injects a `RST` packet
with a broken checksum into the connection. This tricks the GFW into thinking
the connection has been torn down.

In Geneva's syntax, this strategy looks like: `[TCP:flags:A]-duplicate(,tamper{TCP:flags:replace:RA}(tamper{TCP:chksum:corrupt},))-| \/`

TCB Teardowns are not new: they were demonstrated almost a decade ago by [Khattak
et al.](https://www.usenix.org/conference/foci13/workshop-program/presentation/khattak),
and Geneva has discovered [Teardown
attacks](https://geneva.cs.umd.edu/papers/geneva_ccs19.pdf) repeatedly in the
past against the GFW.

Surprisingly, this strategy also can be induced from the server-side.
During the three-way handshake, the server can send a `SYN+ACK` packet with a
corrupt acknowledgement number, thereby inducing the client to send a `RST`.
This causes the `RST` to have an incorrect sequence number (and an
acknowledgement number of 0, but it still is sufficient to cause a TCB Teardown.

**Strategy 4: `FIN+SYN`**

The next strategy appears to be another desychronization attack, but via a
different attack vector. In this strategy, the client (or the server) sends a
packet with the `FIN` and `SYN` flags both set during the three-way handshake.
For the client, in Geneva's syntax: `[TCP:flags:A]-duplicate(tamper{TCP:flags:replace:FS},)-| \/`
For the server, in Geneva's syntax: `[TCP:flags:SA]-duplicate(tamper{TCP:flags:replace:FS},)-| \/`

In the past, we've found the GFW against other protocols has special handling
for `FIN` packets when it comes to resynchronization. In this case, it looks
like the presence of the `FIN` causes the GFW to immediately resynchronize, but
the presence of the `SYN` causes it to think the actual seqno is `+1` from the
actual value, making the GFW off by 1 from the real connection.

We tested this hypothesis by incrementing the sequence number of the actual
request by 1 while this strategy was running, and saw that the client got censored.

From the server-side, the `FIN` flag is not required for this strategy to work.

**Strategy 5: TCB Turnaround**

The TCB Turnaround strategy is simple: before the client initiates the three-way handshake, it first sends a `SYN+ACK` packet to the server. The `SYN+ACK` causes the GFW to confuse the roles of the client and server, thereby allowing the client to communicate unimpeded. TCB Turnaround attacks still work in Kazakhstan, but turnaround attacks do not work against the GFW for any other protocols.

In Geneva's syntax: `[TCP:flags:S]-duplicate(tamper{TCP:flags:replace:SA},)-| \/`

This strategy is client-only, since by the time the `SYN` packet arrives at the server, the censor already knows which side is the client.

**Strategy 6: TCB Desynchronization**

Finally, Geneva identified simple payload-based TCB desynchronization. From the client, injecting a packet with a payload and a broken checksum is sufficient to desynchronize the GFW from the connection. Geneva has identified these in the past against the GFW's censorship of other protocols as well.

In Geneva's syntax: `[TCP:flags:A]-duplicate(tamper{TCP:load:replace:AAAAAAAAAA}(tamper{TCP:chksum:corrupt},),)-|`

This strategy cannot be used from the server-side.

### Summary on Circumvention Strategies

In total, we have discovered 6 strategies that work from the client-side, and 4
that work from the server-side. Each of these works with near 100% reliability,
and can be used to evade the ESNI censorship. Unfortunately, these specific
strategies may not be a long-term solution: as the cat and mouse game progresses,
the Great Firewall will likely to continue to improve its censorship
capabilities.

## Unresolved Questions

It is not yet clear why we observe different durations of residual censorship
from different vantage points.  As with all such research, it is also possible
that there are some regions of China that are affected in different ways than
our vantage points.  If you observe different behavior or that some of our
evasion strategies do not work, please feel free to contact us!

## Thanks

We want to thank all anonymous reviewers who offered us valuable and immediate questions, feedback and suggestions on the [riseup pad](https://pad.riseup.net/p/xCRfphD5CoxmbFcpc1s2).
These comments guided us to prioritize the questions that interest the community the most;
and thus greatly accelerated our research.

We are also thankful to the OONI and OTF community for all of their support.

## Contacts

Geneva team:
* Kevin Bock ([PGP key](https://geneva.cs.umd.edu/keys/kevin_pgp.asc))
* [Dave Levin](https://www.cs.umd.edu/~dml) ([PGP key](https://geneva.cs.umd.edu/keys/dave_pgp.asc))

[GFW Report](https://gfw.report):
* Anonymous ([PGP key](https://gfw.report/gfw_report.asc))
* [Amir Houmansadr](https://people.cs.umass.edu/~amir) ([PGP key](https://people.cs.umass.edu/~amir/Amir%20Houmansadr%20(3C599DC4)%20%E2%80%93%20Public.asc))

We maintain an up-to-date copy of the report on censorship.ai, iyouport.org, gfw.report, net4people and ntc.party.
