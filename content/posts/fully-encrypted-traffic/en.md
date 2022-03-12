+++
title =  "Exposing the Great Firewall's Dynamic Blocking of Fully Encrypted Traffic"
lang = "en"
date = 2021-11-18
tag = []
featured_image = "fiber3.jpeg"
summary = "The Great Firewall (GFW) of China has recently begun blocking fully encrypted traffic in an effort to crack down on circumvention protocols. We demonstrate how this censorship works, what triggers it, how different protocols are affected, and how long residual censorship lasts."
author = "Anonymous, Kevin Bock, David Fifield, Eric Wustrow, Amir Houmansadr, Dave Levin"
+++

Since at least as early as November 6, 2021, [numerous users reported](https://github.com/net4people/bbs/issues/69#issuecomment-962666385) the blocking of their servers running Shadowsocks and VMess+TCP. Outline lead developer Vinicius also [reported](https://github.com/shadowsocks/shadowsocks-libev/issues/2860#issuecomment-974250511) "a drop in the opt-in Outline usage metrics in China starting on November 8". The start of this blocking coincides with the [Sixth Plenary Session of the 19th CPC Central Committee (中国共产党第十九届中央委员会第六次全体会议)](https://zh.wikipedia.org/zh-cn/%E4%B8%AD%E5%9B%BD%E5%85%B1%E4%BA%A7%E5%85%9A%E7%AC%AC%E5%8D%81%E4%B9%9D%E5%B1%8A%E4%B8%AD%E5%A4%AE%E5%A7%94%E5%91%98%E4%BC%9A%E7%AC%AC%E5%85%AD%E6%AC%A1%E5%85%A8%E4%BD%93%E4%BC%9A%E8%AE%AE), which was held from November 8, 2021 to November 11, 2021.

On November 14, 2021, we confirmed that the Great Firewall (GFW) of China has now been able to inspect and dynamically block any seemingly random traffic in real time. This capability potentially affects many censorship circumvention protocols that use encryption to appear random, including (but not limited to) VMess+TCP, Obfs4, and the many variants of Shadowsocks. The censor strategically applies this censorship only against connections from China to certain popular VPS providers, possibly to mitigate over-blocking caused by false positives in traffic classification.

In this report, we demonstrate how the GFW identifies and blocks seemingly random traffic and share code that led to our conclusions. We also offer an effective way to temporarily circumvent the censorship. We then discuss the implication of this blocking incident.

{{% callout color="red" %}}
To maintain reproducibility and promote transparency, we release all the code we used to come to these findings [on Github](https://github.com/breakerspace/madwall-fully-encrypted-traffic-censorship). We also include code in-line throughout this article to help other researchers to reproduce our findings. Note that this code is explicitly designed to trigger (or subvert) this new censorship system, so understand the risks before trying any of the below examples yourself.
{{% /callout %}}

## Our Main Findings

* The GFW can now dynamically block any seemingly random data in real-time, merely based on passive traffic analysis, without relying on its well-known active probing infrastructure.
* The blocking only targets connections from China to a few popular VPS providers outside of China, including Vultr, AlibabaCloud (Hong Kong and Singapore), and Digital Ocean (San Francisco, New York City). Amazon Lightsail, EC2, and Oracle Cloud are reportedly not affected.
* In many cases, after the TCP handshake, a single data packet containing only 1 byte of payload from client to server is sufficient to trigger blocking.
* Only TCP traffic can trigger and is affected by the blocking. UDP traffic cannot trigger and is not affected by the blocking.
* Once triggered, the GFW drops all C2S (client-to-server) **TCP** packets having the same (client IP, server IP, server port) for 120 to 180 seconds. S2C (server-to-client) packets are not dropped.
* The blocking can happen on any port from 1 to 65535.
* While the traffic classification appears to be deterministic, the blocking is probabilistic.
* GFW seems to censor connections whose payload contains less than `70%` printable characters.
* The censor uses protocol fingerprinting to exempt connections from blocking if the first several bytes of the payload match common protocols (including TLS, HTTP, and SSH).
* The censor also allows a connection if its first six bytes are non-whitespace printable characters.

## Background

Tschantz et al. divide approaches to censorship circumvention traffic into two types: *steganograpic* and *polymorphic* (see [Section V and Table 3](https://censorbib.nymity.ch/pdf/Tschantz2016a.pdf#page=8)). The goal of *steganography* is to make circumvention traffic look like allowed traffic; the goal of *polymorphism* is to make circumvention traffic not look like forbidden traffic.

A common way to achieve polymorphism is to fully encrypt the traffic: since fully encrypted traffic presents no plaintext or fixed structures at all, the censor cannot simply identify such traffic with regular expression rules. This is the approach used by Shadowsocks, VMess, Obfs4 and many other censorship circumvention tools. In comparison, most of the protocols that offer encryption, such as TLS, still leave various framing fields unencrypted and therefore can be easily identified; circumvention tools encrypt or otherwise obfuscate these fields.

Fully encrypted traffic is often referred to as "looks like nothing", or misunderstood as "having no characteristics"; however, a more accurate description would be _"looks like random"_. In fact, such traffic does have many characteristics:

* Fully encrypted traffic is [indistinguishable from random](https://en.wikipedia.org/wiki/Distinguishing_attack). One may therefore refer to it as (seemingly) random traffic.
* The data stream possesses *high entropy, homogeneously throughout the entire connection*, starting from the bytes of the first data packets.
* When no packet length obfuscation is implemented, the length of each circumvention packet is equal to `a fixed header length` + `the length of the payload of the proxied traffic`.

## How do we know blocking is happening?

We sent traffic between hosts inside and outside of China. We captured and compared traffic on both endpoints to identify any dropped packets. All experiments were conducted between November 14 and December 8, 2021.

At first, we set up our own Shadowsocks client and server based on [this tutorial](https://gfw.report/blog/ss_tutorial/en/). We then used an automatic script to generate traffic and proxy it through Shadowsocks. The traffic was generated by using curl to visit `http://example.com` and `https://example.com` every 5 seconds. We found that the connections got blocked and unblocked periodically soon after the script started.

Aided by these two observations by Alice et al., we quickly narrowed down sufficient conditions to trigger blocking:

* "[T]he byte streams sent between Shadowsocks clients and servers are, by design, indistinguishable from random" (see [Section 4.1](https://censorbib.nymity.ch/pdf/Alice2020a.pdf#page=6));
* and "[a]fter a TCP handshake, a single data packet from client to server suffices to trigger active probes" (see [Section 4.2](https://censorbib.nymity.ch/pdf/Alice2020a.pdf#page=6)).

To test this, we set up *sink servers* that complete TCP handshakes, but never send data back to the client. We then repeatedly opened connections, sent _random data_ to the server, and closed the connection. Within a few connections, we observed blocking.

## How does the blocking work?

### How can we trigger the blocking?

We observe that a single data packet from client to newly deployed server is sometimes sufficient to trigger the GFW. Often though, we do not observe blocking unless we repeatedly make connections and send random-looking data.

**Reproduce it:**

One can try reproducing the blocking by sending traffic from China to open ports to a server within an affected data center:

1. Start TCP-pinging the port with [Nping](https://nmap.org/nping/):

```
nping -4 -c 0 --tcp-connect $SERVER_IP -p $PORT
```

2. Send 180 bytes of random data to the port (it may not trigger blocking on the first try; you may need to repeat this step a few times to get the port blocked):

```
head -c180 /dev/urandom | nc -vn $SERVER_IP $PORT
```

3. The Nping log shows that the port was blocked right after sending the random data at 147s:

```
...
SENT (144.3590s) Starting TCP Handshake > REDACTED
RCVD (144.5365s) Handshake with REDACTED completed
SENT (145.3615s) Starting TCP Handshake > REDACTED
RCVD (145.5318s) Handshake with REDACTED completed
SENT (146.3639s) Starting TCP Handshake > REDACTED
RCVD (146.5410s) Handshake with REDACTED completed
SENT (147.3660s) Starting TCP Handshake > REDACTED
SENT (148.3671s) Starting TCP Handshake > REDACTED
SENT (149.3693s) Starting TCP Handshake > REDACTED
SENT (150.3715s) Starting TCP Handshake > REDACTED
SENT (151.3736s) Starting TCP Handshake > REDACTED
```

### The same payload does not always trigger the blocking on the first try

We find that while the traffic classification appears to be deterministic, the blocking is probabilistic. That is, sending a payload that *once* triggered the blocking does not always trigger the blocking every time; however, repetitively sending it for a few times will eventually trigger the blocking. On the contrary, sending a payload that *never* trigger the blocking for hundreds of times will not trigger the blocking.

**Reproduce it:**

```sh
# This command triggered blocking. Record the payload in payload file:
head -c180 /dev/urandom | tee payload | netcat -vn $SERVER_IP $SERVER_PORT_1

# Send the same payload again, but it does not always trigger blocking after being replayed:
cat payload | netcat -vn $SERVER_IP $SERVER_PORT_2
```

### Blocking is done by dropping packets from client to server

We captured and compared the packets sent and received on both endpoints. We found that blocking is implemented by dropping C2S (client-to-server) packets. S2C Packets sent by servers are not blocked and are still received at clients.

### No active probing is required before blocking

The GFW makes its blocking decision based purely on passive traffic analysis, without relying on [its well-known active probing infrastructure](https://gfw.report/talks/imc20/en/). We know this because, in most cases, the GFW had not sent any active probes to the server yet before the connection was already blocked.

We want to emphasize that this finding does not mean that [defenses against active probing](https://gfw.report/blog/ss_advise/en) are not necessary or not important anymore. On the contrary, it is because that Shadowsocks-libev, Outline and many other censorship circumvention implementations improved defenses against active probing that forces the censor to strategically take this approach.

Despite this new censorship system, we confirm that the GFW still sends active probes to servers. This evidence warns us that the censor still attempts to accurately identify circumvention servers using active probing.

### Only connections from China to a few well-known hosting services outside of China are affected

We have only been able to trigger the blocking when the servers are hosted by one of a few well-known hosting services outside of China. This finding aligns with [many](https://github.com/shadowsocks/shadowsocks-libev/issues/2860#issuecomment-965999586) [user](https://github.com/shadowsocks/shadowsocks-libev/issues/2860#issuecomment-966340389) [reports](https://github.com/shadowsocks/shadowsocks-libev/issues/2860#issuecomment-966813234), where users could use Shadowsocks servers deployed on Amazon Lightsail, EC2, Oracle Cloud and some "off-brand" VPS providers, but not on AlibabaCloud (Hong Kong and Singapore), and Digital Ocean (San Francisco, New York City).

In particular, we conducted the following testing:

1. Sending random data from a host inside of China to a well-known host provider outside of China: blocking triggered.
2. Using exactly the same pair of hosts in 1, but sending random data from outside-in this time: blocking not triggered.
3. Sending random data from a host inside of China to the open ports of some foreign websites: blocking not triggered.
4. Sending random data from a well-known host provider outside of China to the open ports of some Chinese websites: blocking not triggered.

We also conducted tests from different vantage points within China to different foreign datacenters to confirm our findings.

### A complete TCP handshake is necessary to trigger the blocking

Sending a SYN packet followed by a PSH+ACK packet containing random data (without the server completing its end of the handshake) is not sufficient to trigger blocking. The blocking is thus harder to exploit for [residual censorship attacks](https://geneva.cs.umd.edu/papers/woot21-weaponizing-availability.pdf).

**Reproduce it:**

We tested this with the `test_handshake.py` script, located in the `handshake_tests` folder:

```python3
#!/usr/bin/env python3
"""
A helper script to test if the censorship system can be triggered
with or without a 3-way hanshake.
"""

import sys
import os
import time

from scapy.all import *

dst_ip = sys.argv[1]
dport = int(sys.argv[2])
need_syn_ack = int(sys.argv[3])

seqno = random.randint(1000, 100000)
sport = random.randint(10000, 20000)
syn = IP(dst=dst_ip)/TCP(dport=dport, flags="S", seq=seqno, sport=sport)

ackno = random.randint(1000, 100000)
if need_syn_ack:
    synack = sr1(syn)
    ackno = synack[TCP].seq
else:
    send(syn)
    time.sleep(0.5)

pshack = IP(dst=dst_ip)/TCP(dport=dport, flags="PA", seq=seqno + 1, ack=ackno+1, sport=sport)/Raw(os.urandom(200))
send(pshack)
```

Specifically, we use the following command to 1) send a SYN to an *open* port of the server; 2) wait until we receive a SYN/ACK from the server; 3) and then send a PSH/ACK data packet with 200 bytes random payload. We confirm that it could successfully trigger the blocking:

```sh
sudo ./test_handshake.py $DST_IP $DST_PORT 1
```

We then use the following command to 1) send a SYN to *filtered* port of the server; 2) since the filtered port will not send any SYN/ACK or RST packet, we simply wait for 0.5 seconds; 3) and then send a PSH/ACK data packet with 200 bytes random payload. We confirm that it could **not** trigger the blocking:

```sh
sudo ./test_handshake.py $DST_IP $DST_PORT 0
```

### The blocking happens on all ports

Blocking can happen on all server ports from 1 to 65535. Therefore, running circumvention servers on an unusual port cannot mitigate the blocking.

We tested it by having a sink server listen on all ports from 1 to 65535; then having a small Go program continuously send random traffic to each port until the port became blocked.

**Reproduce it:**

Having a server listen on all ports is not always feasible, since some ports may already be taken. We find the following small trick especially simple and helpful:

1. Let the sink server listen on only one port, for example `12345`.
2. Use iptables to redirect all traffic from `$CLIENT_IP` to the port `12345`: `iptables -t nat -A PREROUTING -i eth0 -p tcp -s "$CLIENT_IP" --dport 1:65535 -j REDIRECT --to-port 12345`.
3. Increase the server process's maximum number of file descriptors: `ulimit -n 655350`.

### UDP traffic does not trigger the blocking

At this time, it appears that this new censorship system is limited to TCP. Because of the absence of UDP blocking, users may experience something interesting when using Shadowsocks: they can still access websites or use apps that use UDP (for example, QUIC), but cannot access websites that use TCP. This is because:

1. Shadowsocks proxies TCP traffic with TCP, and proxies UDP traffic with UDP;
2. and even when the server's port is blocked, UDP packets to or from the same port is not affected.

**Reproduce it:**

To test this, start `nping` in the background:

```sh
nping -4 -c 0 --tcp-connect $SERVER_IP -p $SERVER_PORT
nping -4 -c 0 --udp $SERVER_IP -p $SERVER_PORT
```

Then, run the following simple Python script:

```python
import os
import socket
import sys
import time
s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
s.settimeout(1.0)
while True:
    s.sendto(os.urandom(200), (sys.argv[1], int(sys.argv[2])))
    time.sleep(0.001)
```

Alternatively, one can run:

```sh
while true; do head -c200 /dev/urandom | nc -vn -u $SERVER_IP $SERVER_PORT & done
```

We observed no disruption of `nping` and saw that the UDP packets made it to the server.

### Residual censorship is present

Once the GFW blocks a connection, it continues to drop all **TCP** packets having the same `(client IP, server IP, server Port)` for 120 or 180 seconds. We tested it as follows:

1. Let both Chinese hosts `C1` and `C2` TCP-ping and UDP-ping port `1000` of a foreign host `F1`;
2. Let Chinese host `C1` also TCP-ping and UDP-ping port `1001` of `F1`;
3. Send random data from `C1` to port `1000` of `F1`.

We observed that only TCP connections, not UDP connections, from `C1` to port `1000` of `F1` got blocked. All other pings could still reach the server or get the proper response.

Similar to a past finding on [the GFW's censorship on ESNI traffic](https://github.com/net4people/bbs/issues/43#issue-675228023), we observed duration of residual censorship to be either 120 or 180 seconds. We do not understand why we observed different durations from different vantage points. Unlike [other residual censorship systems](https://geneva.cs.umd.edu/papers/woot21-weaponizing-availability.pdf), the residual censorship timer does not reset when additional packets are sent.


## How does the GFW identify random traffic?

In this section, we describe our understanding of how the GFW identifies fully encrypted traffic. Although we do not fully understand how the traffic analysis algorithm decides whether to block a connection, our experiments led to interesting observations.

### Blocking is not based on entropy

We know that the GFW's existing active probing system likely used entropy (see [Figure 9](https://censorbib.nymity.ch/pdf/Alice2020a.pdf#page=7)) to identify Shadowsocks traffic. We tested if this new system also uses entropy to make a blocking decision.

Initially, our results suggested that entropy might have been used. Messages with very high entropy (such as sending bytes from `/dev/urandom`) trigger blocking quickly, and messages with low entropy (such as a repeated string of `AAAA...`) do not trigger blocking.

However, we find that entropy does not tell the whole story. Some payloads with very high entropy never trigger blocking, whereas other payloads with very low entropy can trigger blocking immediately. 

### Blocking is based on the percentage of unprintable bytes

We theorized that the GFW may be blocking random-looking connections based on the character set in the message, not entropy.

We tested this by writing a script that could generate random messages of a fixed length with a specified percentage of printable characters vs. unprintable characters. Using this, we can test if the censor cares about high frequencies of unprintable messages by slowing increasing the percentage of printable messages within the message, while keeping entropy fixed.

We find that the GFW seems to censor connections if they have less than 70% printable characters.

**Reproduce it:** 

In the open-source repository, see `ascii_tests/generate_ascii_message.py`.

Another easy way to see this in action is by simply `base64`-encoding random bytes and then sending them through the censor. In doing this experiment, we observe no censorship:

```sh
head /dev/urandom | base64 | nc -vn $SERVER_IP $SERVER_PORT
```

Yet, if we send a message with much lower entropy with unprintable characters, blocking occurs almost immediately. 

### The GFW is using protocol fingerprinting to avoid over-blocking

If the GFW is triggering based on unprintable character frequency, it raises the question of why aren't other normal protocols affected? (For example, a TLS handshake consists of only `~27%` printable characters.)

We find that the system is applying protocol fingerprinting to _exempt_ connections from its scrutiny. We reverse engineered the protocol fingerprints used by this system; below, we share the fingerprints. These fingerprints can be used to evade the censor; by simply injecting them into the start of the connection, the connection will be ignored. We note that these protocol fingerprints do not match the fingerprints used by the [Iranian protocol filter](https://geneva.cs.umd.edu/posts/iran-whitelister/).

#### TLS 

We find that if a message starts with the same three bytes as the TLS header, they will be exempted from blocking. Specifically, we find that the protocol fingerprint for TLS is that anything that starts with `\x16\x03` [`\x01` - `\x09`] (so `\x16\x03\x0a` does not fit the fingerprint.)

**Reproduce it:**

For example, while this can trigger the blocking easily:
```
head -c240 /dev/urandom | nc -vn $IP $PORT
```

By simply prepending `\x16\x03\x01` to the front of the message, no censorship will occur:
```
for i in {1..20}; do (printf '\x16\x03\x01'; head -c240 /dev/urandom) | nc -vn $IP $PORT & done
```

#### HTTP Methods

We find that the protocol fingerprint for HTTP GET works by starting messages with an HTTP verb, followed by a space, such as `GET `. We further find that `GET` is case-insensitive: it can be `GeT `, `get `, or other variations. Typos in the verb do not match the fingerprint, however.

Similarly, we find that the protocol fingerprint for HTTP POST works by starting messages with `POST `. The space after `POST` is necessary to evade the blocking. `POST` is also case-insensitive, for example it can be `pOsT `, or other variations. Typos in the verb do not match the fingerprint, however.

We confirm the GFW has fingerprints for `"GET "`, `"PUT "`, `"POST "`, `"HEAD "`. We suspect it also has fingerprints for the other HTTP methods, but we cannot test the others directly. As we will see below, the censor seems to have special handling for messages whose first 6 bytes are all printable characters, and these are the only four HTTP methods that are shorter than 5 bytes. 

**Reproduce it:**

We observe blocking immediately:
```sh
for i in {1..30}; do (printf 'GET'; head -c240 /dev/urandom) | nc -vn $IP $PORT & done
```

We observe no blocking at all:
```sh
for i in {1..30}; do (printf 'GET '; head -c240 /dev/urandom) | nc -vn $IP $PORT & done
```

We observe blocking immediately:
```sh
for i in {1..30}; do (printf 'POST'; head -c240 /dev/urandom) | nc -vn $IP $PORT & done
```

We observe no blocking at all (note the space after the verb):
```sh
for i in {1..30}; do (printf 'pOsT '; head -c240 /dev/urandom) | nc -vn $IP $PORT & done
```

#### SSH

The protocol fingerprint for SSH uses the magic bytes `SSH-[0-9,A-Z,a-z].`
Note that the trailing `.` is required: omitting it fails to match the protocol fingerprint.

**Reproduce it:**

We observe blocking immediately:

```sh
for i in {1..30}; do (printf 'SSH-2'; head -c240 /dev/urandom) | nc -vn $IP $PORT & done
```

We observe no blocking at all:

```sh
for i in {1..30}; do (printf 'SSH-2.'; head -c240 /dev/urandom) | nc -vn $IP $PORT & done
```

The censor may have other protocol fingerprints as well to avoid over-blocking. 

### The GFW exempts blocking if the first 6 bytes of a connection are printable

We observe that the GFW seems to exempt blocking if the first 6 bytes of a connection are printable or spaces (but not other whitespace). 

We tested this with a script that could generate messages where the first _N_ bytes were sourced from different character sets (such as `string.ascii_letters`) and the rest of the message would be random unprintable characters. If the first 6 bytes of the connection are lowercase or uppercase letters (`string.ascii_letters`), digits (`string.digits`), spaces (`" "`), or punctuation (`string.punctuation`), there is no blocking. If there are whitespace characters other than spaces or any other characters in the first 6 bytes, we observe blocking. 

This feature of the censor may be an attempt to limit over-blocking. Even though it may cause the censor to occasionally miss some real random connections, because it uses residual censorship, the censor does not actually need to catch every connection. Since the Shadowsocks makes a new TCP connection for each proxied connection, Shadowsocks client usually sends many TCP connections to the server. As long as one connection triggers the blocking, future connections will be impacted.

**Reproduce it:**

Prepending 5 bytes of printable characters to a random payload will *not* exempt it from blocking:

```sh
(printf 'abcde'; head -c240 /dev/urandom) | netcat -vn $SERVER_IP $SERVER_PORT
```

Prepending 6 bytes of printable character to a random payload will exempt it from blocking. For example, all the following connections were not blocked even after sending them multiple times:

```sh
(printf 'abcdef'; head -c240 /dev/urandom) | netcat -vn $SERVER_IP $SERVER_PORT
(printf 'abcde '; head -c240 /dev/urandom) | netcat -vn $SERVER_IP $SERVER_PORT
(printf 'abcd  '; head -c240 /dev/urandom) | netcat -vn $SERVER_IP $SERVER_PORT
(printf 'abc   '; head -c240 /dev/urandom) | netcat -vn $SERVER_IP $SERVER_PORT
# extreme case where there are only 6 bytes of space characters.
(printf '      '; head -c240 /dev/urandom) | netcat -vn $SERVER_IP $SERVER_PORT
```

### Some 1- and 2-byte payloads can trigger blocking, but not all

We next tested if extremely short payloads could trigger the GFW. We did two experiments:

**Single Byte Experiment**: We enumerated all 1-byte payloads from `0x00` to `0xff`. For each port, every 10 seconds, we 1) made a connection to the port; 2) sent the same 1-byte payload to the port; 3) and then closed the connection. For example, we sent `0x00` to port `50000` and `0x01` to port `50001`. After repeating the experiment many times, we found that 40 payloads out of 256 result in blocking.
We repeated this experiment multiple times.

The bytes that can trigger the GFW are: `'\x0f', '\x17', '\x1b', '\x1d', '\x1e', '\x87', '\x8b', '\x8d', '\x8e', '\x93', '\x95', '\x96', '\x99', '\x9a', '\x9c', '£', '¥', '¦', '©', 'ª', '¬', '±', '²', '´', '¸', 'Ã', 'Å', 'Æ', 'É', 'Ê', 'Ì', 'Ñ', 'Ò', 'Ô', 'Ø', 'á', 'â', 'ä', 'è',` and `'ð'`.

We know the censorship is tied to the payload, not the server port because we used different port ranges in different experiments. We find that the same 40 values always cause blocking. (There was one exception in which we observed timeouts on an additional 13 ports in an experiment, but we could not later reproduce this result.)

Note that the presence of these bytes within a larger message is not enough to trigger blocking all on its own. We tested this by sending different simple messages that start with these bytes (such as `\x1b2345678`). We also found censorship when we tested generating random, fixed-length messages that contained other random bytes besides these 40 bytes. This suggests that these 40 bytes are not fully representative of what the GFW is searching for.

**Double Byte Experiment**: We enumerated all 2-byte payloads whose first byte ranges from `0x00` to `0xff` and whose second byte is `0x0a`. For each port, every 10 seconds, we 1) made a connection to the port; 2) sent the same 2-byte payload to the port; 3) and then closed the connection. For example, we sent `0x000a` to port `50000` and `0x010a` to `50001`. After repeating the experiment many times, we found that 92 payloads out of 256 result in blocking. At this time, we do not understand why more bytes trigger blocking when a newline is appended. 

### Longer payload may more likely to trigger the blocking

We find that longer payloads are more likely to trigger the GFW more quickly. For example, when sending payloads of 2 random bytes, it can take up to 20 connections to trigger blocking.
We also find that it may also depend on the interval when sending: the longer the interval between connections, the more connections are required to trigger blocking.

**Reproduce it:**

```sh
head -c2 /dev/urandom | netcat -vn $SERVER_IP $SERVER_PORT
```

When sending 180 bytes in each connection, usually only 1 to 3 connections are required to trigger blocking:

```sh
head -c180 /dev/urandom | netcat -vn $SERVER_IP $SERVER_PORT
```

## Discussions

This blocking incident reveals many interesting details of the nature of the censor. Understanding this nature will help us 1) better understand the censor's capabilities and limitations, and 2) predict its possible future moves.

### The censor still attempts to avoid over-blocking

A key insight shared by Tschantz et al., after summarizing a large number of real-world censorship incidents, is that "[c]ensors use exploits for which packet loss results in under-blocking instead of over-blocking" (see [Table V](https://censorbib.nymity.ch/pdf/Tschantz2016a.pdf#page=12) and [Recommendation 5](https://censorbib.nymity.ch/pdf/Tschantz2016a.pdf#page=14)).

This conclusion still holds for the current blocking incident, where the censor 1) limits its blocking only to a few popular VPS providers; and 2) uses relatively loose conditions to whitelist protocols.

### The censor's active probing may have been rendered ineffective

The rollout of this new censorship system may be indication that the GFW's active probing infrastructure has become less effective due to recent community efforts. The censor [has been using a combination](https://gfw.report/talks/imc20/en/) of passive traffic analysis, active probing, and potentially human decisions to block Shadowsocks. Such an approach cannot now effectively identify Shadowsocks, thanks to [suggestions by Frolov et al.](https://github.com/net4people/bbs/issues/26) and the patches by developers in the community.

### The censor may still be testing this new system

Our experiments above show that while the censor explicitly tries to avoid targeting innocuous protocols (such as TLS), we suspect, with no evidence, that its traffic classification may have a non-trivial false positive rate. Since the system is effectively a protocol allowlist for unprintable-character-based protocols, any errors in these fingerprints (like [in Iran](https://geneva.cs.umd.edu/posts/iran-whitelister/#dns-fingerprint)) or ommitted protocols will cause collateral damage.

Some of our results also suggest that the censor is not yet fully confident in the new system:

1. The censor limits its blocking to connections from China to a few popular VPS providers. If the censor were confident that its traffic analysis has very few false positives, they could have applied it to *all* connections to the outside of China.
2. The censor does only short-term dynamic port blocking, rather than long term IP blocking. When using this approach, the censor only blocks the port for 120 seconds or 180 seconds. If the censor were confident in having few false positives, it could have blocked the entire IP address of servers for weeks, like what it had done when using its [active probing approach](https://gfw.report/talks/imc20/en/).
3. The censor's blocking rule includes the client IP. If the censor were confident that the server was really a circumvention server, it could have blocked the port or IP of the server for all client IP addresses.

### Censor may tolerate more false positives during politically sensitive times

The start of this less accurate blocking coincides with the [Sixth Plenary Session of the 19th CPC Central Committee (中国共产党第十九届中央委员会第六次全体会议)](https://zh.wikipedia.org/zh-cn/%E4%B8%AD%E5%9B%BD%E5%85%B1%E4%BA%A7%E5%85%9A%E7%AC%AC%E5%8D%81%E4%B9%9D%E5%B1%8A%E4%B8%AD%E5%A4%AE%E5%A7%94%E5%91%98%E4%BC%9A%E7%AC%AC%E5%85%AD%E6%AC%A1%E5%85%A8%E4%BD%93%E4%BC%9A%E8%AE%AE), which may show that censor is willing to tolerate more false positives during politically sensitive times.

## FAQ

### I am a user running a Shadowsocks server. What should I do?

At this time, our recommendation in the short term is to use other datacenters to host the server. Long term, users may need to rely on other circumvention protocols. 

### As a circumvention tool developers, what should I do?

Developer a version with fixed 6 byte of salt/IV; like what suggested by dcf and vinicius.

### Is Tor affected?

This censorship system seems targeted towards Shadowsocks, not Tor. That being said, it shows that the GFW continues to be willing to invest into rendering these circumvention systems ineffective.

### Is obfs4 affected?

https://metrics.torproject.org/userstats-bridge-combined.html?start=2021-01-01&end=2021-12-16&country=cn

At this time based on metrics, `obfs4` doesn't seem to be affected. It depends on how the metrics measured this, because the blocking is dynamic, the first SYN can reach server. Maybe the bridge IPs are mostly not in those IP addresses belong to webhost well-known to Chinese users?

We need to set up our own to check.

### Is Brook affected?

We have not tested Brook directly, but there are anecdotal reports that it is blocked: https://github.com/shadowsocks/shadowsocks-libev/issues/2860#issuecomment-966813234

### If my Shadowsocks server is updated to protect from active probing, am I at still risk?

Although the GFW still sends active probes to suspected servers, it can now conduct dynamic and real-time blocking merely based on passive traffic analysis, without using any information from active probing. In other words, even if the server can defend against active probing attacks, it is still vulnerable to blocking. We want to emphasize that does not mean that [defenses against active probing](https://gfw.report/blog/ss_advise/en) are not necessary anymore.

If your shadowsocks server is updated and accepting connections successfully, you may be running with an IP address the GFW is not yet targeting.

## How can I help?

The current blocking only targets some famous VPS providers, including Vultr, AlibabaCloud, and Digital Ocean. But it possibly does not influence Amazon Lightsail, Oracle Cloud, and others.

Please feel free to let us know of any other cloud vendor that does or does not work.

## Ack

Vinicius Fortuna
Xiangkang Wang
Three anonymous Shadowsocks users who promoptly

