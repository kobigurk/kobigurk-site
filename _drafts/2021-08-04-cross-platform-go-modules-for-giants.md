---
title: StarDrop - anonymous rewards on StarkNet
author: Kobi
comments: true
---

This post is about an experimental project to distribute rewards in a privacy-preserving manner running on StarkNet.

# Introduction

Blockchain projects seek to reward community members who perform activities that are beneficial to the network.

For example, technical users participating in a test network for a project might get a few tokens in return for their participation. Another example includes users reteweeting a few tweets posted by the project.

This brings up a privacy problem. In contrast to normal pseudonymous transactions, these rewards usually require some form of Know Your Customer procedure, even if it's just a twitter handle. From this, it follows that the project awarding the tokens, or in some cases anyone, can track the user's activity after receiving the reward.

We can do better!

StarDrop allows users in a group to claim a reward allocated to them, without the network or the project learning a link between the claims and the users. It uses techniques adapted from [Privacy Pass](https://privacypass.github.io/){:target="_blank"} to build a Verifiable Oblivious Pseudo Random Function. 

The result is a system allowing a project to blindly sign a token that the user can use to redeem a reward later. Moreover, it does so in an on-chain zkrollup, StarkNet, allowing to do this verifiable in scale in low cost, as is appropriate for rewards that are usually not large.

Disclaimer - I've created this project mostly to learn and experiment with StarkNet and it should not be used in production!

# Privacy Pass

Let's first describe a sketch of a simplified version of Privacy Pass that we'll use to build our protocol. In general, Privacy Pass is a 2-party protocol allowing a user to create a blinded token, that a server will sign after performing their checks, and the user afterwards unblinding the signed token and being able to use it without the server learning which token belonged to which user. For a more in-depth description, check out [my post about Privacy Pass](https://kobi.one/2019/01/05/exploring-privacypass.html){:target="_blank"}.

1. A user generates a token $$t$$, which is a string of their choice.
2. The user then hashes it to a curve $$H(t)$$, resulting in an element of a specific elliptic curve.
3. The user chooses a random blinding factor $$b$$ and computes $$T=bH(t)$$.
4. The user then sends $$T$$ to the server.
5. The server uses their private key $$s$$, having public key $$P=sG$$ where $$G$$ is a known generator of the curve, and sign the token, resulting in $$S=sT$$. The server provides a proof that $$S$$ and $$T$$ have the same ratio as $$P$$ and $$G$$, proving that $$S$$ is a valid signed token.
6. The user verifiers the proof and then unblinds it using the inverse of their blinding factor, resulting in $$R=b^{-1}S=b^{-1}sT=b^{-1}sbH(t)=sH(t)$$. Notice that this is still a signed token, just without the blinding factor.
7. The user can then use the signed token $$R$$ to redeem their reward. They do it by sending $$t$$ and $$R$$ to the server.
8. The server then computes by itself $$sH(t)$$, and checks if it equals to $$R$$. If it does, it proves that the user possessed a signed token, as they could not have produced $$R$$ from $$t$$ by themselves.
9. If successful, the server saves $$t$$ as have been redeemed, so it could not be redeemed again.

Note that the original Privacy Pass construction uses an HMAC here, but this detail is not important in our case.

## StarDrop's adaptations

In our case, we'd like to allow users to claim their rewards on-chain, and so there isn't a server we would claim the rewards against. There is only a smart contract. Therefore we have to adapt the protocol in step 7. 

One immediate thought is to put the private key $$s$$ in the contract. That doesn't directly work, since if the key is in the contract then anyone can create their own signed tokens.

Building on that idea, we adapt the protocol to have three phases: commitment phase, key submission phase, redeem phase.

In the commitment phase, users submit their signed tokens $$R$$. In the key submission phase, the server submits their key $$s$$ to the contract. New tokens will not be accepted anymore. In the redeem phase, users can submit $$t$$ and match it with an $$R$$ submitted in the commitment phase. This prevents users from using the published key $$s$$ to create new tokens.

There is another problem which we'll need to handle - front-running. Since now redeeming is all done on-chain, a malicious actor can see the activity a user performs with the contract and front-run both the commitment and the redeem phases. A clever user may notice it, but maybe not. There's an easy way to prevent it though. Make token $$t$$ be a public key, to which the user holds the private key. Then, when redeeming, the contract will verify a signature by that public key. The only thing a front-runner can now do is denial-of-service, but they can't steal a user's token.

The protocol now looks as follows.

1. Initialize the contract with an operator and empty commitment mapping $$C$$ and redeem mapping $$D$$, and start with the commitment phase.
2. Users send their signed tokens $$R$$ from a public key $$u$$. The contract saves in the commitment mapping $$C[u]=R$$.
3. The server then disables further sending of commitments, and moves to the key submission phase.
4. The server then submits its key $$s$$.
5. The users now submit their tokens $$t$$, of which the content is the public key $$u$$, to the contract.
6. The contract computes $$sH(t)$$, checks a signature by $$u$$ on the redeem transaction, and checks whether $$C[u]=sH(t)$$.
7. If successful, the contract transfers the reward to the user's public key $$u$$.

# StarkNet

Now that we have designed StarDrop for an on-chain smart contract, and as a consequence to a zkrollup, we can move on to implement it in StarkNet!

StarkNet is a zkrollup that allows writing smart contracts. This is in contrast to the original deployments of zkrollups that could perform only payments. In that manner, it is similar to zkSync aims to achieve. 

StarkNet works in a model where there's an operator that executes the contracts and creates a proof for them. The user just provides the input. StarkNet uses Cairo under the hood, which is a language that compiles down to VM bytecode a STARK proof can created about.

Cairo supports *hints*, which are a way to provide precomputed inputs to a STARK to make the proof more efficient. These hints are computed using Python programs in Cairo. Running any Python code in StarkNet would then provide a denial-of-service vector against the operator, so StarkNet only allows specific methods to be used with hints, such as signature verification.

Moreover, Cairo is a high level language. There are operations that are significantly more efficient if implemented directly in the low-level AIR representation of the program. Therefore, Cairo uses a few "built ins", such as for performing a range check.

# Implementation of StarDrop in StarkNet

## Hash to curve

The hash $$H$$ takes a string and outputs an element in the elliptic curve with an unknown discrete logarithm against other outputs of the hash. Ideally, it would behave as a good random oracle. Currently in StarkNet, the only hash efficiently available is a Pedersen hash, which does produce a point in the curve, but has algebraic properties that make it visibly different from a random oracle. 

Looking at a simplified version of a Pedersen hash, where the hash of $$b_0b_1$$ is $$b_0G_1+b_1G_2$$, we can give an example. Given the the hashes of $$01$$ and $$10$$, we can compute the hash of $$11$$ without calling the hash function. This is true since $$H(01) = G_2$$ and $$H(10) = G_1$$, and so $$H(11) = H(01) + H(10) = G_1+G_2$$.

This would have to be improved before using this system securely. 

## Scalar multiplication

Being limited to the type of code we can run, we soon stumble into a problem. Step 6 in our protocol requires us to perform a scalar multiplication, and we don't have an efficient method to do it. Can we still do it? The answer is yes!

An easy way is to implement [affine addition and doubling formulas](https://en.m.wikipedia.org/wiki/Elliptic_curve_point_multiplication){:target="_blank"}. For those coming directly from the SNARK world, this would seem problematic - these formulas have exceptions and conditionals are needed unless you can avoid the exceptions. All the branches in a SNARK are always executed, which can become expensive. This is not as big of an issue in StarkNet, since it runs the Cairo VM, and so the programming model is similar to a CPU, where you only pay forthe branch that is actually executing.

An alternative exception-free approach would have been to implement [complete addition and doubling formulas](https://eprint.iacr.org/2015/1060.pdf){:target="_blank"}, but that would likely be more expensive since the number of multiplications needed is larger.

We're still not done - in order to implement the "double and add" method, we need the binary representation of the key. Usually I would do it by providing the binary representation as a hint and checking that when summing it with the relevant powers of 2 we get the scalar. Since we can't use arbitrary hints, we ask the user to provide these hints when claiming the drop. We then check the representation is correct and solely composed of bits. Alternatively, we could ask the operator to provide the representation when submitting their key.

## Contract and scripts

StarDrop is implemented as a StarkNet contract and a collection of Python scripts. The operation is quite manual at this point, but it's fully functional. The implementation [is available on GitHub](https://github.com/kobigurk/stardrop){:target="_blank"}.

The usual workflow looks like (including example invocations):

- The operator initializes the contract using the `initialize` function:

`starknet invoke --address $${ADDR} --abi contract_abi.json --function initialize --inputs 3582577746677722431740023720320633876956623133930419590527228763639578535453 1000 2`

- Users can now submit signed tokens to the contract. To obtain them, they use `blind.py` to create a blinded requests and send it to the operator off-chain. Then the operator uses `sign_token.py` and send the signed blinded token and correctness proofs to the user. The user then runs `unblind.py` to get the unblinded signed token and verify the correctness proofs. Finally, the user sends the commitment to the contract:

`starknet invoke --address $${ADDR} --abi contract_abi.json --function commit --inputs 2042536649993117364296625103999908576248412199423456520000316129072735862954 3261967588085170658862293428587059222438269839015135942205506249609410833918 383622200922475827691447562161872358420401626548600386003465214370732507533 166731635083539346047790634754225714698968525453493336513299480525988223874`

- The operator then ends the commitment phase using `end_commitment_phase.py` and uses the resulting signature:

`starknet invoke --address $${ADDR} --abi contract_abi.json --function end_commitment_phase --inputs 1572514341211381639146744466808246693800547744347028176296700256447030933678 1978481093464076802357781694214950813691007442586456313662641217350746463161`

- The operator submits the key to the contract using `submit_key.py` and uses the resulting signature:

`starknet invoke --address $${ADDR} --abi contract_abi.json --function submit_key --inputs 284697472624386009295839589936418945178 3224560532100339074740113102065711825633396413956247303725126014773482991514 2291167198440858938698896566523731692215085667023695658054610128415429789324`

- Users can now claim their rewards by providing their public key and hints about operator's key and the token:

`starknet invoke --address $${ADDR} --abi contract_abi.json --function claim_drop --inputs 2042536649993117364296625103999908576248412199423456520000316129072735862954 1409864911388159352029816777278381225970523230642148213329717159060539927901 0 1 0 1 1 0 0 1 0 0 0 1 1 1 1 0 0 1 0 1 1 0 0 0 1 1 1 0 0 0 1 0 0 1 1 0 1 1 0 0 0 1 1 0 0 0 0 0 0 1 1 0 0 1 0 1 1 0 1 1 1 1 0 1 1 0 1 0 1 0 1 0 0 1 0 0 1 1 1 0 0 1 0 1 0 1 1 0 1 1 1 1 1 1 1 0 1 1 1 1 1 0 0 0 1 0 1 1 1 1 0 1 0 1 1 1 0 1 0 0 0 1 1 0 1 0 1 1`

- At this point, the user's balance has increased and the pool's balance has decreased. This can be checked:

`starknet call --address $${ADDR} --abi contract_abi.json --function get_balance --inputs 2042536649993117364296625103999908576248412199423456520000316129072735862954`

`starknet call --address $${ADDR} --abi contract_abi.json --function get_pool_balance`

## Limitations

StarkNet Planets, the currently deployed version of StarkNet, is already an expressive zkrollup and fits a lot of our needs, it has a few limitations. An important one is that it doesn't support inter-contract communication in L2 or L1 ↔ L2 communication. 

This means that currently the contract can only track balances that users have internally and not actually transfer token that user can use. The changes needed to make that happen once it does support those are small.

# Conclusion

StarkNet's programmability opens up interesting application possibilities and provides a lot of flexibility as to what can be implemented. The CLI tooling is easy to use and the developers and users don't have to maintain any heavy machinery to create proof or use it. Furthermore, the VS code extension adds useful syntax highlighting.

I did hit a few snags and additionally have some improvement suggestions:

- I managed to kill StarkNet testnet deployment! When using the built in methods, their import order makes a difference, and crashed the StarkNet operator when specified in the wrong order, causing it to stop producing blocks. Kudos to the team that fixed it promptly, and now reject contracts that use the wrong import order. This is reasonable for software in such an early stage and that's what testnets are for 🙂
- The testing workflow is hard. This is because there’s no way to run a StarkNet operator or a simulator of it locally, and so you have to wait for it to process your requests, which can take several minutes. That said, if there are exceptions and not just logic errors, the StarkNet operator reports them early and doesn't wait for the actual execution.
- The Cairo syntax is sometimes overly verbose and sometimes overly implicit. For example, when using built ins, you have to explicitly specify whether a function uses it, but then they’re usually implicitly passed to underlying functions. This was also confusing to me when dealing with "revoked references", having to localize the implicitly passed built ins to prevent that from happening, per Lior's advice.
- Exceptions and asserts can occur at any time and it’s unclear whether an underlying function is safe to call with unchecked inputs. To be fair, this is a problem with most ZKP languages today.
- I haven't put a lot of consideration to costs. Currently, StarkNet doesn't charge you for transaction, and there will be fees in the future. It's likely we'd want to move some of the heavier computation to be paid by the operator.

Overall the experience was great, and I’m looking forward for the next versions!

# Acknowledgements

I thank Louis, Tom and Lior a lot for answering my questions when working on this experiment.
