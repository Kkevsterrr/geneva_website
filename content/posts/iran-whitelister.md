---
title: "Iran: A New Model for Censorship"
summary: "Ahead of its February 21st elections, Iran redeployed its protocol whitelister and mass network degradation, rendering most anti-censorship tools incapacitated or unusably slow. Click read more to learn about how these systems works and how to defeat them."
description: ""
date: 2020-02-26T09:53:38-05:00
draft: true
featured_color: "black"
featured_image: "/laptop.jpeg"
---

**Summary: Ahead of its February 21st elections, Iran deployed two censorship systems: *protocol whitelisting* and *mass degradation,* significantly hindering free communication.** 

- Almost all traffic crossing Iran's borders was degraded. Traffic that was not degraded was matched against a protocol whitelist if destined to certain IP ranges (and any connection that failed to meet the whitelist was dropped).
- These systems rendered almost all existing censorship-evasion tool deployments (VPNs, Tor, Proxies, etc) extremely/unusably slow in Iran.
- Geneva has defeats for these systems: see our Github page [here](https://github.com/Kkevsterrr/geneva).

In this article, we will describe the protocol whitelister, how it works, and how it can be defeated. 

Update: since this was written, the nature of degradation seems to have changed; previously, we could experience degradation across all of our vantage points in Iran, but whitelisted ports were not degraded. Currently, only vantage points in residential or cellular networks are degraded, and whitelisted ports are also affected. The protocol whitelister is still in effect. 

---

{{< figure src="/iran-whitelister-darkmode.svg" class="ph6-l" caption="Test test test" >}}

# Protocol Whitelisting

- Iran's protocol whitelisting system works only over TCP on port 53, 80, 443, and has fingerprints it matches for DNS, HTTP, and HTTPS. Each protocol is not bound to is associated port; the whitelister will match all three protocols on any of the three ports.
- It is not bidirectional. This means that it only affects connections where the client is inside Iran, making it more challenging for researchers to probe and study from outside the country.
- The whitelister monitors the first two packets from the client. If any packet within that time window matches a protocol fingerprint, the flow is unharmed; if no packet does, the second packet and rest of the flow from the client is blackholed. Packets from the server are unharmed, but the client is unable to ACK any data.
- The flow is blackholed by simply dropping all packets from the client after the first packet, while packets from the server are unimpacted (by "flow", we mean packets with the same source and destination IP:port). Although the server can communicate with the client, the client is unable to ACK any data. The whitelister remembers this flow for ~60 seconds after the last matching packet is seen (and each additional packet sent in the flow resets this timer).
- We have observed whitelisting active on port 80 while whitelisting was inactive on port 443. This suggests the whitelister is implemented in multiple distinct systems.
- Not all destination IPs are affected by whitelisting - the whitelister predominantly filters IPs that belong to cloud hosting providers.

In this section we will describe how to trigger the whitelister, which IPs it effects, the fingerprints it looks for, and how it can be defeated.

Our experiments were performed across five vantage points in Iran; four in various networks in Tehran, one in Zanjan to machines we controlled in Amazon EC2 and DigitalOcean. 

## Triggering the whitelister

To trigger the whitelister, you can simply open 53, 80, or 443 on a machine outside of Iran (`nc -lvp 80`) and then connect to it from inside of Iran (`nc <ip> 80`). Simply send a simple message twice (`hi<Enter>hi`) - if the whitelister is running, only the first message will reach the destination server. 

The system is not bidirectional; if the client is located outside of Iran connecting into the country, the whitelister does not interfere with the connection. 

Like Iran's regular censorship systems, the whitelister does not check checksums and is incapable of reassembling TCP segments. 

## Affected IP Space - WIP ****

In order to identify which IPs are affected by the whitelister, we must try to trigger the whitelister to many IPs. The main limitation with the above approach (manually opening a port and sending short text segments) is that control over both client and server is required. Fortunately, we can also trigger the whitelister with normal web traffic by forcing repeated TCP segmentation. TCP segmentation is the process of splitting up the TCP stream across multiple packets. Since the whitelister only verifies the first two packets and cannot reassemble segments, we can segment the request into multiple smaller segments that do not match its fingerprint. Geneva's open-source strategy [engine](https://github.com/kkevsterrr/geneva) allows us to do this. Using this methodology, we can easily test many web servers to see which are affected by the whitelister. 

We performed an experiment to test the effects of the whitelister on the Alex top 20,000 list. To avoid the effects of DNS censorship or accidentally requesting IPs inside of Iran, we used `dig`outside of the Iran to get IP addresses for all 20,000. Inside if Iran, we wet up an experiment with two conditions. The first condition was a control: we made normal `GET` requests to all 20,000 IP addresses using `curl`, and the success or failure of the request was recorded. The second condition tested for the whitelister. We requested all 20,000 IP addresses again, this time with the Geneva engine was running in the background forcing all requests to be repeatedly segmented. IP addresses that we can connect to in the first condition - but time out in the second condition - are potentially affected by the whitelister. We perform this experiment twice to validate the results.

Out of 20,000 IPs, 3,728 IP addresses responded with a timeout during the second condition but responded fine during the first condition in both experiments. Specifically, 4,266 IPs (21.3%) responded differently in the first experiment, 4,564 (22.8%) in the second experiment, with an overlap of 3,728 (18.6%). 

It appears Iran is applying the whitelister at the /24 level. 

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

In this section, we describe the fingerprints matched by each whitelister. By injecting a fingerprint at the start of a connection, the whitelister can be bypassed. Since the whitelister will match any of these fingerprints on all three ports, any fingerprint can be used on any whitelisted ports. 

### DNS Fingerprint

The DNS protocol matcher seems to OK flows if the following conditions are met:

- TCP payload must be ≥ 12 bytes
- Structure of the TCP payload must be a DNS-over-UDP packet, not a DNS-over-TCP packet. This means their whitelister will *never match a legitimate DNS-over-TCP packet.* The reason this does not cause a significant issue is because the whitelister only affects data from the client after the first packet, and there is no second packet from the client in a traditional DNS-over-TCP connection.
- QR must be 0, QDcount must be less than 15, and ANcount must be 0.

Example: `\x00\x00\x01\x00\x00\x01\x00\x00\x00\x00\x00\x00`

### HTTP Fingerprint

The HTTP protocol matcher seems to OK flows if the following conditions are met:

- The TCP payload must be ≥ 8 bytes
- The payload must start with a specific HTTP verb followed by one space; e.g. `"GET "`  or `"POST "`
    - The whitelister can match GET, POST, HEAD, CONNECT, OPTIONS, DELETE, and PUT.
    - TRACE and PATCH do not work.
- A matching packet must be sent within the first two packets.

Example: `GET aaaaa`

### HTTPS Fingerprint

The HTTPS protocol matcher seems to OK the flow if the following conditions are met:

- The TCP payload must be ≥ 41 bytes, (5 bytes for the TLS header, 36 for the TLS Client Hello)
- The length field of the TLS Header equals the length of the Client Hello
- The TLS version header (bytes 2 and 3 of the TCP payload) is TLS 1.0, 1.1, or 1.2,  (`\x03\x01`, `\x03\x02`, `\x03\x03`, respectively). Note that real TLS 1.x Client Hellos all have TLS 1.0 in this field, so this criterion has no practical difference.

After the first 5 bytes of the packet (the type, the version, and the length, 1, 2, and 2 bytes respectively), the whitelister does not look at any contents of the Client Hello. Writing garbage bytes to the remaining bytes of the Client Hello does not trip the whitelister.

Example TCP Payload: `\x16\x03\x01\x02\x00...`, where `\x16` is the indication of a handshake, `\x03\x01` is TLS version (1.0), and `\x02\x00` is the length of the Client Hello (512 bytes)

## Bypassing the Whitelister

We can use Geneva's strategy engine to bypass the whitelister with arbitrary connections by injecting the fingerprint into the connection. Here is a strategy that injects the HTTP fingerprint into the connection stream before each data packet in the connection:

`[TCP:flags:PA]-duplicate(tamper{TCP:load:replace:GET%20aaaaaaaaaa}(tamper{TCP:chksum:corrupt},),)-| \/`

We can use it as follows:

    $ S="[TCP:flags:PA]-duplicate(tamper{TCP:load:replace:GET%20aaaaaaaaaa}(tamper{TCP:chksum:corrupt},),)-| \/"
    $ sudo python3 engine.py --strategy "$S" --server-port 80 --log info &
    $ nc <ip> 80
    hi
    hi
    # connection is still successful
    

In this case, we used Geneva to defeat the whitelister for a simple netcat application, but we can also use this to defeat whitelisting and run a non-degraded VPN on port 80. 

---

# Collateral Damage

Much censorship evasion design today is couched in this idea of maximizing *collateral damage* for the censor. The idea is that if a censor wants to shut down an anti-censorship tool, we should make it as costly as possible for them to do so, usually by designing the system in such a way that it forces censors to block much more than they need to - causing collateral damage. For example, running bridges/proxies/VPNs on EC2 has been popular under the assumption that a censor would have to block all of AWS in order to shut them down, which would cause enormous collateral damage due to all the websites that are hosted on AWS.

The whitelister represents a way for Iran to sidestep some of this collateral damage. By only allowing content from specific protocols, they can shutdown or degrade most anti-censorship tools without negatively impacting regular websites or services.  

# Conclusion

Censoring nations have greater capacity for censorship than they exercise on a daily basis, and often these systems can be highly effective at crippling existing censorship-evasion tools (VPNs, Tor, etc).  The unidirectional and non-universal nature of the whitelister makes it more challenging for researchers outside of the country to identify it. 

# Citations

[1] Protocol whitelisting was briefly described in 2013: [https://censorbib.nymity.ch/pdf/Aryan2013a.pdf](https://censorbib.nymity.ch/pdf/Aryan2013a.pdf), as a capability they turned off after the 2013 elections were over.

[2] [https://www.ndss-symposium.org/wp-content/uploads/2019/02/ndss2019_03B-2-1_Frolov_paper.pdf](https://www.ndss-symposium.org/wp-content/uploads/2019/02/ndss2019_03B-2-1_Frolov_paper.pdf)
