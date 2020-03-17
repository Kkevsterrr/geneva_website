---
title: "Iran: A New Model for Censorship"
summary: "Ahead of its February 21st elections, Iran redeployed its protocol whitelister and mass network degradation, rendering most anti-censorship tools incapacitated or unusably slow. Click read more to learn about how these systems works and how to defeat them."
description: ""
date: 2020-02-26T09:53:38-05:00
draft: true
featured_color: "black"
featured_image: "/laptop.jpeg"
---

**Summary: Ahead of its February 21st elections, while Iran degraded internet traffic crossing its borders, it subtly deployed a secondary censorship system: a *protocol whitelister.*** 

- The whitelister only allows a small list of protocols to be used, posing a threat to almost all existing censorship-evasion tool deployments (VPNs, Tor, Proxies, etc).
- Due to its design, this system is challenging to detect, measure, or study from outside the country.
- We deployed *Geneva* against the protocol whitelister and have discovered multiple ways to defeat it.

In this article, we will describe the protocol whitelister, how it works, and how it can be defeated. As of time of writing, the whitelister is still in effect from all of our vantage points within Iran. 

All of our experiments were performed across five vantage points in Iran; four in various networks in Tehran and one in Zanjan to machines we controlled in Amazon EC2 and DigitalOcean. 

---

# Deployment

Ahead of the elections in Iran, Iran began degrading internet traffic crossing its borders, and this caught the attention of many in the online community within Iran.  

{{< center-me >}}
{{< tweet 1229834253866807298 >}} 
{{< tweet 1234162649866362885 >}} 
{{< tweet 1233844197276315648 >}} 
{{< /center-me >}}

However, while internet degradation was taking place, Iran subtly deployed a **secondary censorship system**, a *protocol whitelister*, which garnered much less attention. The protocol whitelister only affects traffic leaving Iran and is minimally invasive to most non-forbidden traffic, making it harder for researchers to detect (in fact, we discovered the whitelister by accident while studying Iran's normal censorship systems.)

---

# Protocol Whitelisting

Iran's protocol whitelisting system affects only TCP traffic on port 53, 80, 443, and only to specific IP ranges. The whitelister works by performing deep packet inspection (DPI) on the packet payloads sent by clients in Iran; if the data sent at the start of the connection does not match an allowed protocol fingerprint, the rest of the flow from the client is dropped. The whitelister has fingerprints to match for DNS, HTTP, and HTTPS traffic, but each protocol is not bound to is associated port: the whitelister will match all three protocols on any of the three ports.

The whitelister is challenging to detect for several reasons. 

- It is not bidirectional. This means that it only affects connections where the client is inside Iran, making it more challenging for researchers to probe and study from outside the country.
- The server-side has no indication censorship has taken place. Once the whitelister steps in to censor the traffic flow, it only affects the packets leaving the client: packets from the server are unharmed. Although the client will still get the server's data, the client is unable to ACK or respond to any data.

At the start of a connection, the whitelister monitors the first two packets from the client. If either of those two packets matches a protocol fingerprint, the flow is unharmed; if no packet does, the second packet and rest of the flow from the client is blackholed. Once triggered, the whitelister remembers a flow for ~60 seconds after the last matching packet is seen (and each additional packet sent in the flow resets this timer).

{{< figure src="/iran-whitelister-darkmode2.svg" class="ph6-l" >}}

The system works in tandem with Iran's standard censorship system: the protocol whitelister ensures communication only occurs over certain protocols that the regular censorship system can verify.

In our experiments with the whitelister, we identified that the whitelister is not all-or-nothing: we have observed whitelisting active on port 80 while whitelisting was inactive on port 443.

Curiously, not all destination IPs are affected by whitelisting. Of the Alexa top 20,000 list, approximately 20% of the sites were hosted on IP addresses affected by the whitelister. 

## Triggering the whitelister

To trigger the whitelister, we can open port 53, 80, or 443 on a machine outside of Iran (`nc -lvp 80`) and then connect to it from inside of Iran (`nc <ip> 80`). Sending a simple message twice  (`hi<Enter>hi`) is sufficient to be affected by the whitelister - if the whitelister is running and affects the destination IP, only the first message will reach the server. 

Like Iran's regular censorship systems, the whitelister does not check checksums and is incapable of reassembling TCP segments. 

## Affected IP Space ****

In order to identify which IPs are affected by the whitelister, we performed an experiment to test the effects of the whitelister on the Alexa top 20,000 list. To avoid the effects of DNS censorship or  requesting IPs inside of Iran (thereby not crossing the whitelister), we used `dig` outside of Iran to get IP addresses for all 20,000. 

Inside of Iran, we set up an experiment with two conditions. The first condition was a control: we made normal `GET` requests to all 20,000 IP addresses using `curl`, and the success or failure of the request was recorded. The second condition tested for the whitelister. We requested all 20,000 IP addresses again, this time sending `G`, `E` and `T` in separate messages. 

IP addresses that respond in the first condition but time out in the second condition are likely affected by the whitelister. We perform this experiment twice to validate the results.

Out of 20,000 IPs, 3,728 IP addresses responded with a timeout during the second condition but responded fine during the first condition in both experiments. Specifically, 4,266 IPs (21.3%) responded differently in the first experiment, 4,564 (22.8%) in the second experiment, with an overlap of 3,728 (18.6%). 

To determine ownership of these IPs, we performed reverse DNS lookups on each.

    1540 amazonaws.com
     159 your-server.de
     158 akamaitechnologies.com
      66 linode.com
      27 poneytelecom.eu
      16 vultr.com
      15 cloudfront.net
       8 hwclouds-dns.com
       6 ovh.net
       5 scaleway.com

<more info here>

## Fingerprints

In this section, we describe the fingerprints matched by each whitelister. Knowing these fingerprints can be a powerful tool for evaders: by injecting a fingerprint at the start of a connection, the whitelister can be bypassed. Since the whitelister will match any of these fingerprints on all three ports, any fingerprint can be used on any whitelisted ports. 

### DNS Fingerprint

The DNS protocol matcher seems to OK flows if the following conditions are met:

- TCP payload must be ≥ 12 bytes
- Structure of the TCP payload must be a valid DNS-over-UDP header, not a DNS-over-TCP header. This means their whitelister will *never match a legitimate DNS-over-TCP packet.* The reason this does not cause a significant issue is because the whitelister only affects data from the client after the first packet, and there is no second packet from the client in a traditional DNS-over-TCP connection.
- Specific limitations exist for the fields within the DNS header: QR must be 0, QDcount must be less than 15, and ANcount must be 0.

Example: `\x00\x00\x01\x00\x00\x01\x00\x00\x00\x00\x00\x00`

### HTTP Fingerprint

The HTTP protocol matcher seems to OK flows if the following conditions are met:

- The TCP payload must be ≥ 8 bytes
- The payload must start with a specific HTTP verb followed by one space; e.g. `"GET "`  or `"POST "`
- The whitelister can match GET, POST, HEAD, CONNECT, OPTIONS, DELETE, and PUT. TRACE and PATCH do not work.

Example: `GET aaaaa`

### HTTPS Fingerprint

The HTTPS protocol matcher seems to OK the flow if the following conditions are met:

- The TCP payload must be ≥ 41 bytes: 5 bytes for the TLS header, 36 for the TLS Client Hello
- The length field of the TLS Header equals the length of the Client Hello
- The TLS version header (bytes 2 and 3 of the TCP payload) is TLS 1.0, 1.1, or 1.2,  (`\x03\x01`, `\x03\x02`, `\x03\x03`, respectively). Note that real TLS 1.x Client Hellos all have TLS 1.0 in this field, so this criterion has no practical difference.

After the first 5 bytes of the packet (the type, the version, and the length, 1, 2, and 2 bytes respectively), the whitelister does not look at any contents of the Client Hello. Writing garbage bytes to the remaining bytes of the Client Hello does not trip the whitelister.

Example TCP Payload: `\x16\x03\x01\x02\x00...`, where `\x16` is the indication of a handshake, `\x03\x01` is TLS version (1.0), and `\x02\x00` is the length of the Client Hello (512 bytes)

## Using Geneva to Bypass the Whitelister

Geneva (*Gen*etic *Eva*sion) is a genetic algorithm that *evolves* censorship evasion strategies against a censor. Unlike most anti-censorship systems, it does not require deployment at both ends of the connection: it runs exclusively at one side (client or server) and defeats censorship by manipulating the packet stream to confuse the censor without impacting the underlying connection. A *censorship evasion strategy* describes how that traffic should be modified. Since Geneva will be evolving these strategies, they are expressed in a domain-specific language that comprises the DNA of each strategy. For a full rundown of Geneva's strategy DNA syntax, see [our Github page](https://github.com/kkevsterrr/geneva).

Geneva is comprised of two main components. First, the genetic algorithm, which can evolve new ways to defeat a censorship system given an application that experiences censorship and a fitness function. Second, the strategy engine, which applies a given strategy on the fly to modify active network traffic. 

We wrote a simple fitness function for the whitelister and deployed Geneva against the protocol whitelister. *In under two hours*, it discovered two simple strategies that defeat it (beyond simply injecting an innocuous HTTP request). In this section, we will explore these surprisingly simple strategies. 

The Geneva strategy engine is open source on [our Github](http://github.com/kkevsterrr/geneva), so all of these strategies can be deployed and used by anyone. Since they operate at the TCP layer, they can be applied to any application: with Geneva running, even an unmodified web browser can become a simple censorship evasion tool. To learn more about how Geneva (or the Geneva strategy engine) works under the hood, see our [papers](http://censorship.ai/papers) or [about](http://censorship.ai/about) page. 

Note that Geneva is *not* designed as a general purpose evasion tool, and does not provide any additional encryption, privacy, or protection. It is a beta research system and it is not optimized for speed. Use these strategies at your own risk. 

### Strategy 1

The first strategy works by simply sending one additional packet before the 3-way handshake: an empty packet with the `PSH/ACK` flags set. 

In Geneva's strategy DNA syntax, this strategy looks like this:

`[TCP:flags:S]-duplicate(tamper{TCP:flags:replace:PA,)-|`

This triggers on all outbound TCP `SYN` packets and duplicates them; to the first duplicate, the TCP `flags` field is set to `PSH/ACK`, and the second duplicate is unchanged. Both packets are sent on the wire. When the packets arrive at the server, the server ignores the `PSH/ACK` packet, since it is not associated with any connection yet, but the whitelister processes it, causing it to ignore the rest of our connection. The `SYN` arrives as normal and our connection can start unchanged. 

We do not understand why this strategy works, though we hypothesize the `PSH/ACK` packet tricks the whitelister into thinking it has already missed the relevant data packets, causing it to ignore the rest of the flow.

### Strategy 2

The second strategy is stranger than the first. This strategy works by sending *nine copies* of the `ACK` packet during the 3-way handshake.

In Geneva's strategy DNA syntax, this strategy looks like this:

`[TCP:flags:A]-duplicate(duplicate(duplicate,duplicate),duplicate(duplicate,duplicate(duplicate(duplicate,),)))-|`

By sending nine ACKs during the 3-way handshake, the whitelister ignores the rest of our packets. We hypothesize this works because the whitelister has some internal limit on the number of packets it processes for a given flow, and by sending these 9 packets, we can exceed this total, causing the whitelister to ignore the rest of our connection. 

### Strategy 3

Beyond exploiting the whitelister's shortcomings at the TCP layer, we can also use Geneva's strategy engine to bypass the whitelister by simply injecting one of the protocol fingerprints into the start of the connection. Here is one such strategy that injects the HTTP fingerprint into the connection stream before each data packet in the connection:

`[TCP:flags:PA]-duplicate(tamper{TCP:load:replace:GET%20aaaaaaaaaa}(tamper{TCP:chksum:corrupt},),)-| \/`

### Using a strategy

To deploy one of these strategies, we can use Geneva's strategy engine (open source on [our Github page](https://github.com/kkevsterrr/geneva)) to apply these strategies to our network traffic. Since the engine captures network traffic on a specified port, any application sending data on that port will be affected by the strategy. For example, we can test that these strategies work by trying to trigger the whitelister with `nc` as above with the engine running in the background.  

    $ STRATEGY="[TCP:flags:PA]-duplicate(tamper{TCP:load:replace:GET%20aaaaaaaaaa}(tamper{TCP:chksum:corrupt},),)-| \/"
    $ sudo python3 engine.py --strategy "$STRATEGY" --server-port 80 --log info &
    $ nc <ip> 80
    hi
    hi
    # connection is still successful
    

In this case, we used Geneva to defeat the whitelister for a simple netcat application. We can also use this to defeat whitelisting and run a VPN or similar system on port 80. 

---

# Conclusion

Much censorship evasion design today is couched in this idea of maximizing *collateral damage* for the censor. The idea is that if a censor wants to shut down an anti-censorship system or block a resource, we should make it as costly as possible for them to do so. Usually this is done by designing the system in such a way that it forces censors to block much more than they want to, causing collateral damage. For example, running bridges/proxies/VPNs on EC2 has been popular under the assumption that a censor would have to block all of AWS in order to shut them down, which would cause enormous collateral damage due to all the websites that are hosted on AWS.

The whitelister represents an attempt for Iran to sidestep some of this collateral damage. By only allowing content from specific protocols, they can shutdown or degrade most anti-censorship tools without negatively impacting regular websites or services.  

Iran has had a greater capacity for censorship than they have exercised in the past, and these systems can pose a threat to existing deployments of censorship-evasion tools (VPNs, Tor, etc). Worse, the unidirectional and non-universal nature of the whitelister makes it more challenging for researchers outside of the country to identify and study. With tools like Geneva, we hope to shorten the response time for the censorship evasion community and alert tool developers how they can make their tools more resilient in the ever-changing censorship landscape. 

---

# Citations

[1] Protocol whitelisting was briefly described in 2013: [https://censorbib.nymity.ch/pdf/Aryan2013a.pdf](https://censorbib.nymity.ch/pdf/Aryan2013a.pdf), as a capability they turned off after the 2013 elections were over.

[2] [https://www.ndss-symposium.org/wp-content/uploads/2019/02/ndss2019_03B-2-1_Frolov_paper.pdf](https://www.ndss-symposium.org/wp-content/uploads/2019/02/ndss2019_03B-2-1_Frolov_paper.pdf)
