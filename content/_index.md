---
title: "Geneva: Evolving Censorship Evasion"
featured_color: "red"
description: "Join us and learn about our fight to defeat censorship around the world."
---

{{< padded >}}
{{< flex >}}
{{% flex-column %}}

# Automating Evasion

Researchers and censoring regimes have long engaged in a cat-and-mouse game, leading to increasingly sophisticated Internet-scale censorship techniques and methods to evade them. In this work, we take a drastic departure from the previously manual evade/detect cycle by developing techniques to **automate the discovery of censorship evasion strategies**.

{{% /flex-column %}}

{{% flex-column %}}

# Our Approach

We developed Geneva (*Gen*etic *Eva*sion), a novel experimental genetic algorithm that evolves packet-manipulation-based censorship evasion strategies against nation-state level censors. Geneva re-derived virtually all previously published evasion strategies, and has discovered new ways of circumventing censorship in China, India, Iran, and Kazakhstan.

{{% /flex-column %}}

{{< /flex >}}


{{< center-me >}}
{{% markdown %}}

# How it works 

Geneva runs _exclusively_ on one side of the connection: it does not require a proxy, bridge, or assistance from outside the censoring regime. It defeats censorship by modifying network traffic on the fly (by injecting traffic, modifying packets, etc) in such a way that censoring middleboxes are unable to interfere with forbidden connections, but without otherwise affecting the flow. Since Geneva works at the network layer, it can be used with any application; with Geneva running in the background, any web browser can become a censorship evasion tool. Geneva cannot be used to circumvent blocking of IP addresses.  

Geneva composes four basic packet-level actions (drop, duplicate, fragment, tamper) together to represent _censorship evasion strategies_. By running directly against real censors, Geneva's genetic algorithm evolves strategies that evade the censor. 


{{% /markdown %}}

{{< /center-me >}}
{{< /padded >}}

{{< section src="/flags_noborder.png" >}}

{{< flex >}}

{{% flex-column %}}

# Real World Deployments

{{% /flex-column %}}

{{% flex-column %}}
Geneva has been deployed against real-world censors in China, India, Iran, and Kazahkstan. It has discovered dozens of strategies to defeat censorship, and found previously unknown bugs in censors. 

All of these strategies and Geneva's strategy engine and are open source: check them out on our [Github page](http://github.com/kkevsterrr/geneva).

Learn more about how we designed and built Geneva [here](/about).

{{% /flex-column %}}

{{< /flex >}}

{{< /section >}}

{{< padded >}}
{{< center-me >}}
{{% markdown %}}

# Who We Are
This project is done by students in [Breakerspace](https://breakerspace.io), a lab at the University of Maryland dedicated to scaling-up undergraduate research in computer and network security.

This work is supported by the Open Technology Fund and the National Science Foundation.

# Contact Us

Interested in working with us, learning more, getting Geneva running in your country, or incorporating some of Geneva's strategies into your tool? 

The easiest way to reach us is by email.
 - Dave: dml (at) cs.umd.edu (PGP key [here](/keys/dave_pgp.asc))
 - Kevin: kbock (at) terpmail.umd.edu (PGP key [here](/keys/kevin_pgp.asc))
{{% /markdown %}}

{{< /center-me >}}
{{< /padded >}}

