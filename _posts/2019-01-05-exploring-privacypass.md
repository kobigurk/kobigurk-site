---
title: Exploring Privacy Pass
author: Kobi
comments: true
---

This post explores Privacy Pass, a protocol which "lets users prove their identity across multiple sites anonymously without enabling tracking". We will go through the protocol components and eventually see a [fully-compatible implementation of Privacy Pass in Rust](https://github.com/kobigurk/privacypass-rs){:target="_blank"}.

## Privacy in the internet

Privacy for users in the internet is often advocated for, and just as often, compromised. This happens for good reasons - creating social networks (i.e., Facebook), allowing smarter usage of our data (i.e., Gmail), convenience of usage (i.e., OAuth - "Login using Google"), Security (i.e., Cloudflare's HTTPS-for-every-site and Denial-of-Service protection) and many other reasons.

The privacy loss is usually not absolute - it allows privacy against some adversaries. For example, while Facebook has access to your personal data, it takes the responsibility of not exposing it to other users, and does it pretty well.

It makes me glad when these services which potentially compromise privacy also take additional steps to re-introduce it in creative ways.

## Cloudflare's Crypto Week
One of the companies whose trade-offs I like is Cloudflare. I've been using their service for many years to easily introduce HTTPS to my websites without having to go through the usual complex processes of obtaining and managing an HTTPS certificate.

A few weeks ago, Cloudflare had a [Crypto Week](https://blog.cloudflare.com/tag/crypto-week/){:target="_blank"}, where each day they posted about newly-developed features, who use cryptography to improve their services, in aspects such as privacy.

Although not part of the Crypto Week, this exposed me to a protocol Cloudflare supports since 2017 - [Privacy Pass](https://blog.cloudflare.com/cloudflare-supports-privacy-pass/){:target="_blank"}. This protocol has been developed in collboration with academic researchers and is provided as a service by Cloudflare.

## Privacy Pass

The Privacy Pass protocol, [as implemented in Cloudflare](https://captcha.website/){:target="_blank"}, attempts solving the following tension:

Cloudflare would like to give access to websites to humans rather than, let's say, mal-intentioned bots. To do that, Cloudflare requires users to solve a [CAPTCHA](https://en.wikipedia.org/wiki/CAPTCHA){:target="_blank"} when they have suspicion.
On the one hand, these challenges protect the websites - no non-humans may access the site. On the other hand, false-positives cause inconvenience to users. That puts the burden on Cloudflare to improve their human-detection capbalities without compromising on security too much.

This is where Privacy Pass comes in. It allows a user to solve a CAPTCHA once and use this "proof-of-humanity" later on.

## The protocol

Rather than outlying the protocol directly, I'd like to build it step-by-step, to highlight the importance of each of the components in the protocol, and why stopping at that level might not be enough.

### Methodology

Methodology-wise, I believe the best way to understand a topic is being able to teach it. In teaching, there are multiple levels. Two of these are teaching another human (outlying the theory, proofs, math) and teaching a machine (implementing it so it can be used).

While teaching another human is highly valuable, some aspects are more readily exposed and understood when teaching a machine. This is one of the reasons I've chosen to [implement Privacy Pass in Rust](https://github.com/kobigurk/privacypass-rs){:target="_blank"} - in this way I could see the nitty-gritty details of how Cloudflare and the researchers made Privacy Pass work in practice.

### Detecting a human

Cloudflare uses CAPTCHAs to detect humans when suspicion arises. Essentially, Cloudflare presents a challenge with weird-looking letters and numbers:

![CAPTCHA](/assets/images/captcha.jpg)

The user then provides the solution to what they think is the solution. If Cloudflare approves it, they allow the user through.

Recall that the goal of Privacy Pass is to carry this "proof-of-humanity" to other situations where the user might be suspected.

So why not, let's say, when Cloudflare receives a correct solution, make Cloudflare send multiple "passes" which can be used in other sites?

The answer lies in *tracking*. When Cloudflare would send these passes to the user, they will know that these passes are related to the original CAPTCHA solution, and by so, to the original website where the user saw the challenge.

If Cloudflare would have liked, they could track the user across multiple sites. To show a concrete example:

* The user, being at home, would have gone to kobi.one, see a challenge and solve it, let's denote the solution $$S$$. They would send the solution to Cloudflare, containing the solution $$S$$ and natually their home IP.
* Cloudflare would approve the solution, and send passes $$P_1, ... P_N$$. Additionally, they would store in their database the connection between $$S$$, each $$P_i$$ and the user's IP.
* Next, when the user uses one of the passes in other websites, maybe from a coffee shop having a different IP, Cloudflare would be able to correlate the usage of pass $$P_i$$ with the user.

Cloudflare could obviously *not* store this data, and that would require putting the trust in Cloudflare. While they might deserve it, you can do better.

### Discerete Logarithm Problem

What if Cloudflare could provide the passes without being able to track their usage forward? Apparently, with some [elliptic-curve cryptography](http://en.wikipedia.org/wiki/Elliptic_curve_cryptography){:target="_blank"}, it is possible.

Without diving too much into elliptic-curves, I'll remind some basic facts.

Elliptic curves, being curves, have points. You can define the $$+$$ operation on points, such that given points $$P$$ and $$Q$$ on the curve, you will get a third point $$R$$ which is the result of $$P+Q$$. To see how to define this visually, you can see it, for example, [here](https://www.certicom.com/content/certicom/en/21-elliptic-curve-addition-a-geometric-approach.html){:target="_blank"}.

You can also define adding a point $$G$$ to itself $$n$$ times ($$G+...+G$$ with $$n-1$$ additions). Call this operation $$n \cdot G$$.

You can ask, given points $$G$$ and $$Y$$, "how many times would I have to add $$G$$ to itself in order to get $$H$$?" That is, what is the $$n$$ such that $$n \cdot G = H$$? This is a hard problem in elliptic-curve cryptography, called the [Discrete Logarithm Problem](https://en.wikipedia.org/wiki/Discrete_logarithm){:target="_blank"}.

### Blind signatures

Now we begin solving the problem of tracking. First, let's expand our protocol and describe it in more detail.

Let's say the secret key of the Privacy Pass server is the number $$x$$ and there's a publicly known point on the curve called $$G$$ and execute the protocol as follows:

* The user, requesting passes, will create a random token $$t$$ and send the server the point $$t \cdot G$$, denoting it $$T$$. The server will not be able to know $$t$$ because of the Discrete Logarithm Problem.
* The server will create the point $$x \cdot T$$. Let's call it a *signature* and denote it $$Z$$. The user will not be able to uncover $$x$$ because of DLP.
* The server will send the signature to the user, which later on will be able to use this signature to prove they received it from the server. Since only the server knows $$x$$, only the server could create $$Z = x \cdot T$$, and so the server will know they have created these signatures.

This has the problem of tracking, since the server will be able to store all this data and correlate it to the user when the $$Z$$s is used by the user.

The user, instead of sending $$T$$ directly can use a *blinding factor* $$r$$ to create $$M = r \cdot T = r \cdot t \cdot G$$. We call this a *blinded token*. Now, when receiving the signature $$Z$$ over each $$M$$, the user will *unblind* the signature by multiplying by the inverse of $$r$$: $$r^{-1} \cdot Z = r^{-1} \cdot x \cdot M = r^{-1} \cdot x \cdot r \cdot T = r^{-1} \cdot r \cdot x \cdot T = x \cdot T$$. Let's denote this result $$N$$.

Although the user could cacluate $$N$$, they could never generate it by themselves - since they don't know $$x$$. Presenting the server $$t$$ and $$N$$ allows the server to verify they indeed the ones who generated this signature, and the server will not know which *blinded token* this signature relates to since they never knew the *blinding factor* $$r$$.

The protocol now looks like this:

* The user, requesting passes, will create a random token $$t$$ and a blinding factor $$r$$ and send the server the point $$r \cdot t\cdot G$$, denoting it $$M$$.
* The server will create the point $$x \cdot M$$. The server generates a signature $$Z$$.
* The server will send the signature to the user, and the user unblinds the signature, calculating and storing $$N=r^{-1} \cdot M$$.
* The user later on presents $$t$$ and $$N$$ to the server when they would like to prove their humanity.

It sounds very good! Unfortunately, this doesn't solve the problem of tracking, which might not be immediately obvious.

Assuming the server uses the same secret key $$x$$ for all the users requesting passes, this works well. What if the server uses a different $$x$$ for every user? We're right back to square one - the server, when verifying the signature, will be able to iterate over all the different $$x$$s they generated and thus identify the user.

This is why it's important for the server to show it indeed uses the same secret key $$x$$ for all the users. This is done by using a *Discrete Logarithm Equality Proof*.

### Discrete Logarithm Equality Proof

The server already publishes one point publicly - $$G$$. The server could publish publicly an additional point $$Y=x \cdot G$$. This is essentially a *commitment* to the secret key $$x$$.

The construction of how *DLEQ* works requires a bit more math, and I prefer not getting into it in this post. That said, I would like to give a few points to those interested:
* It is similar to the Schnorr identification scheme, proving that the server knows the secret key $$x$$ - essentially proving knowledge of "$$Y/G$$".
* It uses the non-interactive variant of the scheme using the Fiat-Shamir heuristic, which also has an [IETF RFC](https://tools.ietf.org/html/rfc8235){:target="_blank"}, converting it from an *honest-verifier zero-knowledge* scheme to a *non-interactive zero knowledge argument*.
* Instead of only proving knowledge of $$Y/G$$, it also shows that the *same* proof applies to $$Z/M$$. This means that the publicly committed secret key, as shown in $$Y$$, is the *same* one used to create $$Z$$ for $$M$$, proving that the server used the same secret key to generate the signature.

A nice description of *DLEQ* and some of these details are described in [this post by Matthew Green](https://blog.cryptographyengineering.com/2017/01/21/zero-knowledge-proofs-an-illustrated-primer-part-2/){:target="_blank"}.

If you would like more details on the math inolved, feel free to [reach out](https://kobi.one/contact.html){:target="_blank"}.

### Finalizing the protocol

By requiring *DLEQ*, the server now can't cheat and use different secret keys to track users. To make this protocol usable in practice, some other aspects should be considered:

* Instead of sending a single token, the user could send multiple tokens to be signed at once.
* The server should limit the amount of tokens the user can request for each CAPTCHA solution, to prevent abuse by a user dispensing these tokens to other users, essentially making the protection useless.

## Implementation

The original open-source implementation of Privacy Pass by Cloudflare and the researchers is [available here](https://github.com/privacypass){:target="_blank"}.

[My implementation of Privacy Pass in Rust is available here](https://github.com/kobigurk/privacypass-rs){:target="_blank"}. Some details about it:

* It is fully-compatible with the [reference implementation](https://github.com/privacypass){:target="_blank"}, both as a server and as a client.
* It uses [amcl](https://github.com/milagro-crypto/amcl){:target="_blank"} for elliptic-curve cryptography.
* It uses [RocksDB](https://github.com/rust-rocksdb/rust-rocksdb){:target="_blank"} for storage.
* It is probably compilable to WebAssembly or other architectures with minor modifications.

Additionally, it is easy to modify it to support any kind of challenge-solution use-case: login to systems, voting and many more.

## Conclusion

Privacy Pass allows adding privacy to a system where initially it seems impossible, by the use of cryptography. It's awesome to see that some companies, which have all the means to disregard privacy at the cost of convenience, still go through the R&D and risk required to re-introduce privacy to their systems.

If you found this interesting and have more questions or suggestions, feel free to [reach out](https://kobi.one/contact.html){:target="_blank"}, I'd love to hear from you!
