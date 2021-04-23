---
title: "Geneva Code Release"
summary: "We're excited to announce the full code base of Geneva is now public! Click to read more about the code, how to use it, and why we are releasing it."
description: ""
date: 2020-07-12T09:53:38-05:00
draft: false
featured_color: "black"
featured_image: "concert.jpeg"
---

# Genetic Algorithm Release

We're happy to announce that the full code base of Geneva is now [public](https://github.com/kkevsterrr/geneva). Please read through our [documentation](https://geneva.readthedocs.io) thoroughly and understand the risks before trying it out. 

{{% callout color="red" emoji="" %}}

**Disclaimer**: Running Geneva or Geneva’s strategies may place you at risk if you use it within a censoring regime. Geneva takes overt actions that interfere with the normal operations of a censor and its strategies may be detectable on the network. During the training process, Geneva will trip censorship many times. Geneva is not an anonymity tool, nor does it encrypt any traffic. Understand the risks of running Geneva in your country before trying it. If you have any questions or concerns, please feel free to [contact us](https://geneva.cs.umd.edu/people/#contact-us).

{{% /callout %}}

# What does this code do?

Our code base is comprised of two key components:

- **Genetic Algorithm:** The logic underlying Geneva is its genetic algorithm, which comprises multiple "packet manipulation" primitives (duplicate, tamper, fragment, and drop) to form sophisticated censorship evasion strategies. The genetic algorithm trains against *live* censors to discover new evasion strategies. If you wish to discover new evasion strategies, this is the code to use.
- **Engine:** The Geneva "engine" executes packet manipulation strategies by capturing and modifying inbound and outbound on the fly. The genetic algorithm uses the engine while training against live censors, but the engine does not need the genetic algorithm. If you simply wish to apply known evasion strategies, this is the code to use. The engine uses our domain-specific language to specify packet manipulation strategies; you can copy-paste the strategies directly from our [papers](/papers/) (the code also includes many examples).

# What should I do with new evasion strategies?

If you run our genetic algorithm and discover new evasion strategies, we welcome you to share them with us! We are very interested in learning about new strategies—where they work, against what censors, for what protocols, etc. Sharing your findings with us can help lead to new discoveries about censorship around the world—and we will be happy to work with you to ensure your safety in sharing these results with us.

To share new results with us, the easiest way to reach us is by email. 

- Kevin: kbock (at) umd.edu (PGP key [here](https://geneva.cs.umd.edu/keys/kevin_pgp.asc))
- Dave: dml (at) cs.umd.edu (PGP key [here](https://geneva.cs.umd.edu/keys/dave_pgp.asc))

# Why Release the Code?

Any time a new technique for censorship evasion is developed, researchers must ask themselves if it is ethical to release the code. Censoring nation-states watch what new tools, papers, and techniques are released: if we made Geneva open-source, couldn't nation-states use Geneva themselves to identify bugs and patch them before evaders can use them? 

We believe the net effect of releasing Geneva helps evaders.

The first consideration is resources: our [research papers](/papers/) and talks detail how we built, tested, and deployed Geneva. Nation-state adversaries have the resources to re-implement Geneva themselves regardless of whether we release the code or not. To this end, releasing the code democratizes its use: regardless of the resources available to you, Geneva presents an automated solution to identifying bugs in censors.

The second consideration is the nature of the bugs Geneva finds themselves, and how it finds them. Geneva gives us the ability to *fast forward* the censorship arms race. Techniques that previously could take weeks or months to find can now be identified in hours or days. With this in mind, we ask: *what is the logical conclusion of the censorship arms race?* 

Geneva generally finds two classes of strategies: bugs in implementation and systemic logic issues. We believe that minor implementation bugs may be easier for censors to simply patch. However, issues that are more systemic to the design and limitations of the middleboxes, such as those that attack state tracking, are likely more expensive for censors to fix. Long-term, we expect the minor implementation bugs to be patched but systemic issues to remain.

Only time will tell how nation-state censors will engage in the arms race with a tool like Geneva. In the meantime, we will continue to advance the state of the art in automating censorship evasion to help evaders open communication everywhere.
