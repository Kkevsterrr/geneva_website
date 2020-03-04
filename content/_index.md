---
title: "Geneva: Evolving Censorship Evasion"
featured_color: "red"
description: "Join us and learn about our fight to defeat censorship around the world."
---

{{< padded >}}
{{< flex >}}

{{% flex-column %}}

# Censorship Evasion

Researchers and censoring regimes have long engaged in a cat-and-mouse game, leading to increasingly sophisticated Internet-scale censorship techniques and methods to evade them. In this work, we take a drastic departure from the previously manual evade/detect cycle by developing techniques to **automate the discovery of censorship evasion strategies**.

{{% /flex-column %}}

{{% flex-column %}}

# Our Approach

We developed Geneva (*Gen*etic *Eva*sion), a novel genetic algorithm that evolves packet-manipulation-based censorship evasion strategies against nation-state level censors. Geneva re-derived virtually all previously published evasion strategies, and has discovered new ways of circumventing censorship in China, India, Iran, and Kazakhstan.

{{% /flex-column %}}

{{< /flex >}}


{{< center-me >}}
{{% markdown %}}

# How it works 

Geneva runs _exclusively_ on one side of the connection: it does not require a proxy, bridge, or assistance from outside the censoring regime. It defeats censorship by modifying network traffic on the fly (by injecting traffic, modifying packets, etc) in such a way that censoring middleboxes are unable interfere with forbidden connections, but without otherwise affecting the flow. Since Geneva works at the network layer, it can be used with any application; with Geneva running in the background, any web browser can become a censorship evasion tool. 

Geneva composes four basic packet-level actions (drop, duplicate, fragment, tamper) together to represent _censorship evasion strategies_. By running directly against real censors, Geneva's genetic algorithm evolves strategies that evade the censor. 


{{% /markdown %}}

{{< /center-me >}}
{{< /padded >}}

{{< section src="/flags_noborder.png" >}}

{{< center-me >}}
{{% markdown %}}
# Real World Deployments

Geneva has been deployed against real-world censors in China, India, Iran, and Kazahkstan. It was discovered dozens of strategies to defeat censorship. 

Since Geneva often identifies many strategies that defeat a censor, we created a _taxonomy_ to classify them.
 - _Species_ describe the overarching bug in a censor that Geneva exploits to defeat censorship
 - _Subspecies_ captures unique ways that Geneva exploits the bug
 - _Variants_ are unique strategies that use the same attack vector, but do so differently

Against China, India, and Kazahkstan, we describe 4 species (two of which are novel), 8 subspecies, and 21 variants in our first paper. See our [Github page](https://github.com/kkevsterrr/geneva) for more information on how to use these.  
{{% /markdown %}}

{{< /center-me >}}
{{< /section >}}
