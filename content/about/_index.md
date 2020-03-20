---
title: "About Geneva"
shorttitle: "About"
weight: 1
featured_color: "purple"
description: "How it Works"
---

{{% markdown %}}
# What is Geneva? 
Geneva is a novel experimental genetic algorithm that evades censorship by manipulating the packet stream on one end of the connection to confuse the censor. Geneva is comprised of two components:
its _genetic algorithm_ and _strategy engine_. The strategy engine runs a given censorship evasion strategy over active network traffic; the genetic algorithm is the learning component that evolves new strategies (using the engine) against a given censor.

{{% /markdown %}}


{{< flex >}}
{{% flex-column %}}
# Genetic Building Blocks


{{< flex >}}
{{% nopad-flex-column %}}
{{< figure src="/duplicate.svg" >}}
{{< figure src="/tamper.svg" >}}
{{% /nopad-flex-column %}}

{{% nopad-flex-column %}}
{{< figure src="/fragment.svg" >}}
{{< figure src="/drop.svg" >}}
{{% /nopad-flex-column %}}
{{< /flex >}}

{{% /flex-column %}}
{{% flex-column %}}
Geneva is a genetic algorithm, a form of biologically-inspired artificial intelligence. Much like how biological systems compose simple building blocks (the A, T, C, and G of DNA), Geneva generates new algorithms by composing very basic ways of manipulating packets. It can duplicate, tamper, drop, or fragment packets. Geneva composes these individual actions into "action trees."

Action trees have a _trigger_; this controls which packets the action trees act upon. These action trees together form censorship evasion strategies.

{{% /flex-column %}}
{{< /flex >}}

{{< flex >}}
{{% flex-column %}}

Geneva creates many random individual strategies and runs each of them against real censors. Based on how successful they are (and other factors), it assigns a numerical fitness to each individual. The most fit survive from one generation to the next, and Geneva mutates and mates strategies to create new ones. 

The key to Geneva's success is its fitness function, which encourages the genetic algorithm to explore the space of strategies that does not damage the underlying TCP connection.

Over many successive generations, if Geneva discovers a strategy that defeats censorship, the fitness function encourages the genetic algorithm to refine and simplify the strategy. 

{{% /flex-column %}}
{{% flex-column %}}

# Composition

{{< figure src="/tree.svg" height="300">}}

{{% /flex-column %}}
{{< /flex >}}
{{< flex >}}
{{% flex-column %}}

# Evolving Evasion
{{< figure src="/segmentation.svg" height="300" caption="">}}

{{% /flex-column %}}
{{% flex-column %}}

Unlike prior work, we do not manually create censorship evasion strategies: Geneva finds them automatically. Then, _after_ Geneva finds how to evade censorship, we analyze its output to learn more about how censors work. This has led to interesting insights, like how China's great firewall synchronizes its state or how it processes packets. 

Sometimes, like with the strategy to the left, we conclude that Geneva has found a bug in the censor. This strategy is an example of the _Segmentation_ species Geneva discovered in China against the Great Firewall by segmenting the forbidden request twice at specific indices. Though the Great Firewall can normally reassemble segments, by segmenting twice at these bounds, we can trick the Great Firewall into ignoring our request to evade censorship.

{{% /flex-column %}}
{{< /flex >}}

{{% markdown %}}
To read more about how Geneva works, see our [papers](/papers). 

{{% /markdown %}}
