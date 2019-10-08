---
layout: post
title: Introducing BitSniff
---

On September 5th-6th, during the [Bitcoin emBassy Hackathon](https://www.meetup.com/bitcoin-il/events/264327474/), myself and [Michael Maltsev](https://m417z.com/) developed **BitSniff** - a tool for detecting Bitcoin-related communications in encrypted traffic. Today we release an updated and more stable version of it. You can check the interactive [demo](https://m417z.com/bitsniff/) or clone the GitHub [repository](https://github.com/m417z/bitsniff) to use it yourself.  
The following is the project write-up, focused on motivation and methodology.

![_config.yml]({{ site.baseurl }}/images/bitsniff/front.png)

## Motivation
As Bitcoin stubbornly continues to exist and gain traction, it also starts grabbing attention of those in charge. With censorship resistance being one of Bitcoin's key value propositions, the ability of third parties to establish who participates in this new economy may become a non-negligible issue. Imagine Bitcoin getting completely outlawed in China - being detected while using it may have very grim consequences.  
Therefore, we must ask ourselves - who else knows you are running Bitcoin?

For most nodes in the network, the answer is "pretty much everyone", given that Bitcoin P2P communications aren't even encrypted and the same port is being used almost universally. But, arguably, most nodes in the network don't care much about being detected.  
Now for those who do care, some steps are obvious - using VPN and running the node over Tor being the most obvious ones. We aim to show that it may not be enough, by implementing a technique that is able to detect Bitcoin communications using nothing but traffic volume over time - an information even most privacy concerned individuals are likely leaking to their law-abiding Internet Service Provider.

## Background
Every time you use software that interacts with a Bitcoin network, and especially a Bitcoin node, you leave a sticky fingerprint in your traffic. It comes in the form of a small, but unavoidable spike in volume every time a new block is mined and the nodes start gossiping about it. The blocks in Bitcoin are quite big, and the propagation speed is critical for consensus (greater delay means more frequent accidental forks), so such effect is predictable, and, in a sense, inherent to the Bitcoin architecture.  
Notably, the volume of block-related messages was drastically reduced since the introduction of Compact Block Relay ([BIP 152](https://github.com/bitcoin/bips/blob/master/bip-0152.mediawiki)). Instead of requesting whole blocks, mostly consisting of transactions already known to the node, the peer informed of a new block is only requesting the missing transactions. Yet the amount of extra communications in the seconds following a new block is still considerable.  

![_config.yml]({{ site.baseurl }}/images/bitsniff/protocol-flow.png)

_(Image taken from the BIP 152 page)_

This effect may not be noticeable for a single block, but over time it gets statistically significant, and may get exploited.

## Methodology
Our goal, given a time series of traffic volume over time, is to determine whether it tends to have larger volume just after a new block is found, during a window of typical propagation delay. We also aim to provide a meaningful metric measuring how confident we are that Bitcoin communications are, indeed, present.

![_config.yml]({{ site.baseurl }}/images/bitsniff/flowchart.png)

* An input file is parsed to create a target time series, aggregated in units of 1 second for further calculations speed-up.
* Using the earliest timestamp and the length of the target, the actual block times during that window are fetched. Note that this information is public by design.
* Block times are transformed into expected activity time series by adding bell-like shapes after block times, sized according to typical propagation delay.

We now have two time series, the target and the expected. How similar they are? How confident we are that this level of similarity is not accidental?

* To answer the first question, we calculate the correlation of the target and the expected. The problem is, the correlation is not a sufficient metric by itself, as what correlation we should consider meaningful depends on the shape of the traffic (e.g. 40% may be very significant for some shape of the traffic, and really low for others).
* We address that issue by generating a lot of fake expected time series that have, on average, the same number of blocks as actual one, and calculate the traffic correlation to every one of them. Those fakes are very similar to the expected activity, except for one thing - the block times are selected randomly, regardless of the real block times. If the target traffic is as similar to these as it is to real block times, this similarity is meaningless.
* We calculate the z-score of the actual correlation compared to the fake ones - a measure of how far the actual correlation is from the fakes average, compared to how scattered the fakes tend to be. Intuitively, we now know not only how similar our traffic is to the expected Bitcoin activity - we also know how unlikely such similarity was to occur by chance.
* Most people probably feel more comfortable with percentages than with z-scores, so we finish the process with approximating the corresponding percentage confidence level using the Z table.

All that is left is to define a threshold for tagging a traffic as Bitcoin-related. There is no right value per se, and it can be determined empirically to achieve the desirable balance between true positive and false positive performance.

## Performance
The performance of the attack is a function of traffic length, with longer logs corresponding with better performance. For the ease of presentation, we used a single confidence threshold of 95%, preferring to err on the false negative side.  
For the true positive estimation we used our own full node traffic, logged for 24 hours. Note that we did record on the 8333 port, so the results apply to dedicated nodes only. We will discuss mixed traffic in a later section.  
There are infinitely many options to define false positive. We mostly used, arguably, the 'hardest' one - the same actual full node traffic, but with shifted timestamps (e.g. shifted three hours backwards). This way, the traffic logs still represent Bitcoin activity, but the logs don't match the real block times. We also added several YouTube traffic logs. None of that did matter much as with a given threshold the false positive rate was, effectively, zero.

![_config.yml]({{ site.baseurl }}/images/bitsniff/performance.png)

## Use Cases
As mentioned above, one use case for the technique is detecting Bitcoin nodes by governments or ISPs. Our primary motivation behind this project is raising awareness regarding this possibility.  
It can also be applied to many other blockchain-based currencies. Bitcoin forks with bigger blocks are an even easier target, and so are currencies with higher block density, such as Monero, Litecoin and Ethereum (assuming someone ever succeeds to run an Ethereum node).

The technique has a more marketable use as well - detecting illegal mining activity. Most mining software uses variations of the Stratum protocol, that naturally has activity associated with new blocks on the underlying blockchain - the pool has to distribute a new block template for the miners, losing profits until it is done.  
One common case is gaining illicit access to electricity for Bitcoin mining, exploiting corporate resources or governmental facilities. Another is Monero CPU mining malware. Currently, most antivirus software relies on binary signatures and known endpoints to detect mining malware, both of which can be tricked.

## Protection
There are many ways to go about it, but staying completely undetected is far from trivial - traditional privacy enhancing tools mostly focus on the packet level, which is orthogonal to the technique. Let’s break up the potential defence vectors.
 
* **VPN / Tor** - unlikely to affect the time series shape much, and therefore for larger traffic lengths the statistical significance of block-related spikes will inevitably become overwhelming.
* **Traffic mixing** - for traffic volumes that are orders of magnitude higher than Bitcoin P2P communications, mixing is likely to be very effective. That would, however, demand constant shielding of both upstream and downstream communications, and couldn’t be done effectively by just running the node on a general purpose machine - any noticeably long unshielded period may be enough for detection.
* **Being your own ISP** - too spicy for most, but that should work.

Beyond active measures available now, both privacy and bandwidth efficiency of Bitcoin communications are actively worked on. It is entirely possible that the messaging protocol will get to the point where block propagation doesn’t trigger any significant spikes in traffic volume.

## Hackathon Pitch
<style>.embed-container { position: relative; padding-bottom: 56.25%; height: 0; overflow: hidden; max-width: 100%; } .embed-container iframe, .embed-container object, .embed-container embed { position: absolute; top: 0; left: 0; width: 100%; height: 100%; }</style><div class='embed-container'><iframe src='https://www.youtube.com/embed/9S8xsDq3PTU' frameborder='0' allowfullscreen></iframe></div>

## Acknowledgements  
The technique idea was originally investigated in work by:
* [Prof. Ittay Eyal](http://webee.technion.ac.il/people/ittay/)
* [Prof. Amir Houmansadr](https://www.cics.umass.edu/faculty/directory/houmansadr_amir)
* [Fatemeh Rezaei](https://www.linkedin.com/in/fatemeh-rezaei-8a745940/) 
* [Niko Kudriastev](https://79jke.github.io/)
