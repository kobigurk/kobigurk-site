---
title: Cryptocurrency recovery (or "What the BIP is all of this?")
author: Kobi
excerpt: Having Bitcoins is nice and having them being stored securely is even nicer. Though, as often is the case, the additional security often invites complexity to frequent use of your money. There are great software and hardware solutions that make your money safer than ever. But it's not always easy to recover.
---
# Overview
## TL; DR
### What I can help with
* Got any type of coin stuck for which you saved some backup and want to consult whether they can be rescued? Contact me - [kobigurk@gmail.com](mailto:kobigurk@gmail.com)!
* Want me to walk you through the recovery process? Gladly! 
* Want to consult how to properly backup your private keys in order to be able to recover your coins in the future? Contact me :-)
### How to recover by yourself
* Got Bitcoin Cash or Bitcoin Gold stuck in an address whose private key you have? Use [bcash-instadump](https://github.com/shesek/bcash-instadump)
* Got Bitcoin Cash or Bitcoin Gold stuck in an Electrum wallet whose 24 words seed you saved? Use the [BIP39 tool](https://iancoleman.io/bip39/) to derive the private keys and then use [bcash-instadump](https://github.com/shesek/bcash-instadump)
* Got Bitcoins stuck in a Ledger Nano S who was destroyed but you have the 24 words saved? Import them into Electrum 
* Got any type of coin stuck in a Ledger Nano S who was destroyed but you have the 24 words saved? Use the [BIP39 tool](https://iancoleman.io/bip39/) to get your private keys and use their respective wallets to import them
* Got Bitcoin Cash or Bitcoin Gold stuck in a multi-sig address in Electrum/Copay/anything, but have access to your Bitcoins? It's a bit more complex, but possible - read through the deep dive or contact me - I can help!


Want to know more about the technical details? Keep reading.

# Deep dive
Having Bitcoins is nice and having them being stored securely is even nicer. Though, as often is the case, the additional security often invites complexity to frequent use of your money. There are great software and hardware solutions that make your money safer than ever.

What's nice about these solutions is that they hide the complexity of securing your money and give you the best-practice approach to do it. The problem is that they deal with the specific use-case they had in mind and are less flexible when you'd like to deviate a bit from the usual path.

For example, let's say you use [Electrum 2FA (Two-Factor Authentication)](http://docs.electrum.org/en/latest/2fa.html). It's a great solution that uses [TrustedCoin](https://api.trustedcoin.com) as a co-signer to each of your transactions. Quoting from them, 

>With a two-factor setup, your computer cannot spend your bitcoins without your entering a code from your cell phone. You will have significantly increased security.

That's great! And there's also a backup key that allows you to recover your Bitcoins in case TrustedCoin ever goes offline, so you're safe. But...

Recently some forks in Bitcoin happened. There's Bitcoin Cash, Bitcoin Gold and maybe some more precious metal variants in the near-future. Those variants operate on a different chain than the usual Bitcoin, and that means that Electrum and TrustedCoin do not support it.

Damn.

If both my wallet and the co-signing service do not support Bitcoin Gold... I'm stuck! They hid all the complexity of how they secured me and now I don't know how to recover.

That was just an example of a situation that can happen in many other use-cases - hardware wallets (such as [Ledger Nano S](https://www.ledgerwallet.com/r/6a0a?path=/products/ledger-nano-s) which I absolutely recommend), multi-sig wallets such as [Copay](https://copay.io/) and others.
 
It's important to stress that usually the developers of the different platforms are doing an awesome job - they give you all the necessary bits of data that enable recovery. What they don't do, understandably, is supporting every different recovery method that a user would like to have. 

For example, if your Ledger Nano S hardware wallet is destroyed because you were trading while taking a bath, the main recommendation is to buy a new one and use the "seed words" you were given when you created your wallet. That's great, but what if Ledger goes bankrupt? I have to rummage through ebay to find rare Ledgers? Meh. 
  
How to store Bitcoins securely is a different matter I won't get into right now, but I want to tell you some **good news**: most of the wallets I mentioned here and many others use similar methods to store your coins, and that means that most of the steps of recovery are the same. 

In this post I will survey the preliminaries to understand how wallets store your Bitcoins (and other coins) using a single seed, how you would recover your Bitcoins in case you decide (or can) not to use a specific service anymore and what is the backup data you should keep in order to facilitate that.

# Preliminaries
Bitcoin development happens through BIPs. These are Bitcoin Improvement Proposals. They are announced on the mailing list and assigned a number. Roughly, if there's strong support behind it, it's implemented in Bitcoin clients. 

Following is a high-level description of some of the BIPs that support the mechanism with which different wallets manage private keys.

## BIP32
[BIP32](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki) describes a standard way to generate multiple addresses and private keys from a single private key, in a secure way. The exact technical details can be read in the BIP itself, but I'll highlight the ones I feel important:

* A parent (extended) private key can generate many distinct children private keys by an index.
* The derivation can be done in a "normal" way or a hardened way, where a significant security difference exists - in the normal way, it's possible (with some additional meta-data) to retrieve the parent private key from a child private key, but you gain an interesting feature - given the parent **public** key, you can view all of its normal children public keys and, for example, monitor their balance. 
* The way you represent this derivation is by a **path** - something like "m/0'/0". This path means - take the (extended) private key, take its hardened child 0 and then from that resulting private key take the normal child 0 - and then you gain a private key you can use.

In drawings it looks like this:
![path](/content/images/2017/12/SoWkIImgAStDuSfLqBLJC53dKZ9GLm8pkHnIyrA0CW00.png)
translates to
![path translated](/content/images/2017/12/Untitled.png)


## BIP39
[BIP39](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki) describes a standard way to generate a private key from a sequence of human-readable words.

From the user's perspective, she has a Mnemonic Sentence - a sequence of [words](https://github.com/bitcoin/bips/blob/master/bip-0039/bip-0039-wordlists.md) from which a binary array is derived. The exact number of words is related to the amount of entropy which translates to the level of security. Ledger Nano S, for example, uses a 24 word sequence.

There's an awesome [tool](https://iancoleman.io/bip39/) by [Ian Coleman](https://github.com/iancoleman) that allows you to generate the master private key given your mnemonic sentence, and following that, the children private keys derived from that. You can even run this tool offline by either compiling it yourself, or saving the html and running it on an offline computer.


## BIP44
[BIP44](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki) uses BIP32 and describes a standard way to generate private keys such that you can use the same master private key for different cryptocurrencies or use-cases without risk.

That is, it standardizes the path that is used by BIP32:
```
m / purpose' / coin_type' / account' / change / address_index
```


* purpose - constant **44'**, to signify that this is a BIP44-compatible derivation.
* coin_type - different coins have different numeric types. Bitcoin is 0 and there seems to be consensus that Ethereum is 60, Litecoin is 2 and many others. Ian Coleman's tool has an extensive list of these, but a wallet doesn't have to conform to it - the coin type is just a convention.
* account - maybe you are managing wallets for different users or purposes. This element in the path allows you to distinguish them by assigning different indices to them.
* change - is this an address that receive money, or a change addresses that just received change resulting from money I sent to someone? Can be 0 or 1.
* address_index - the index of the address in this account, because you're using a new address in each transaction when possible... I hope!

The literal meaning is - you take the master private key, you go to the purpose-indexed hardened child, the coin\_type-indexed hardened child of it, the account-indexed hardened child of that, the change-indexed normal child of this and finally the address_index-indexed normal child of those.

## BIP16
[BIP16](https://github.com/bitcoin/bips/blob/master/bip-0016.mediawiki) is a bit more complex to describe in a few words that also convey the motivation, but I'll try - it describes a general way to encode complex scripts in a way that's compatible with single-address schemes and moves the burden of paying the fee for these complex scripts from the sender to the receiver.

Such a complex script can be M-out-of-N multi-sig - an address from which you can only spend using M signatures out of N parties involved. This mechanism is used, for example, in Electrum 2FA, with a 2-out-of-3 multi-sig.

Bitcoin's scripting language is rich and I gave a talk on how it works and other interesting scripts you can use in the [BlockchainWTB conference](https://drive.google.com/open?id=0B-EU9txb0XfwYkE1eFFVLW1tLU0).

# Recovery
## Types of addresses
Now we're getting to the juicy stuff! We now know all the ingredients we need to recover from many different situation and I'm to describe some of these through concrete examples.

### P2PKH (regular addresses)
When you use your new Ledger Nano S for the first time, it walks you through a setup process. In this process, you are presented with a mnemonic sentence of 24 words. 

Now you know that these words actually generate a master private key for your Ledger using BIP39.

When you use your different wallet in the chrome apps that Ledger supplies you with, you are actually using the same master private key but with different coin types, account indices and address indices, as specified in BIP44.

Even more, the awesome guys at Ledger give you the option to see the exact path to your addresses - just go to your account settings in one of the wallet apps, and you will see a path that looks like a BIP44 path - "m/44'/0'/0".

So, if you want to recover from situation where Ledger doesn't work anymore, you could import it into Electrum, for example. 

Let's assume, though, you'd like to know for 100% you don't depend on any single wallet software to exist anymore - you can use the BIP39 tool and see the private keys there and generate transactions using these private keys using any Bitcoin wallet you'd like. That can be [Bitcoin Core](https://bitcoin.org/en/download) or even a Bitcoin library such as [BitcoinJS](https://bitcoinjs.org/).

### P2SH (multi-sig)
P2SH is BIP16 - roughly, the method to use complex scripts such as multi-sig, as used by Electrum 2FA, Copay and others.

Recovering from a multi-sig wallet service going offline is a bit more complex, but it's still possible.

Let's examine a 2-out-of-3 multi-sig Bitcoin wallet case. In this case, the address is derived from the **3** **public** keys of the different participants. Let's assume we have these:

* A primary key which you own and use in the usual transactions
* a backup key which you store offline 
* the key of a wallet protection service, such as TrustedCoin

To transact from this address, you'd have to sign the transaction using **2** of the **private** keys. In this case, you'd usually use your primary one and the wallet protection service would provide the second signature. In case the wallet protection service goes offline, you'd use your primary and backup key.

The thing is, that in order to transact from a P2SH address you don't need just the 2 private keys, but also a **redeem script** derived from the 3 public keys. That means, that in order to successfully recover, you also need this redeem script. So make sure you save that as well.

## OK, fine... How to actually do it?
For normal addresses, an ideal recovery software would roughly do this:

* Use BIP39 with your seed words to recover your master private key
* Collect the unspent outputs of the address you want to spend from - these are the Bitcoins you'll send
* Sign a transaction that sends money from this address to a destination address for your choosing

An example of such tool that automates a lot of this process in the case of Bitcoin Cash and Bitcoin Gold forks is [bcash-instadump](https://github.com/shesek/bcash-instadump) by Nadav Ivgi. 

For P2SH addresses, you'd have to do the same, but when you sign the transaction, you'd provide multiple private keys and the redeem script. This redeem script is easily attainable in good wallets such as Electrum. I had such a case and wrote some code that uses exactly this redeem script to extract Bitcoin Gold using BitcoinJS.

For other types of coins that use this scheme the recovery process is similar in theory, but it usually requires a different set of libraries and wallet software. For instance, in Ethereum, you may use [MyEtherWallet](myetherwallet.com) with the derived private keys.

If you have some stuck Zcash, you may notice that the BIP39 tool misses it. That's actually because of a technical limitation in BitcoinJS - Zcash use an address prefix which is 2 bytes, but BitcoinJS only supports a single-byte address prefixes. I have a hacky-fix from it that I use, but it's not generic enough to publish.

For some coins (such as Bitcoin Gold), there are also some wallets that supports the import of single-address private keys, but are not open-source. That doesn't immediately mean the wallet developers are malicious, but it bugs me to use something proprietary like this for sensitive matters like money.

# Conclusion

I hope this post would encourage people to take advantage of the best security practices, such as using a hardware wallet or multi-sig. 

In this latest price surge, a lot of new folks come in, techie or otherwise, and are worried of using complex software, but using the easier-to-use ones (such as hardware wallets) feels like being dependent on a specific piece of software. It might feel like you're losing control of your coins, contrary to the promise of cryptocurrencies.

I hope this post showed you that you're not actually strongly tied to it - you could always recover from it, given you that you backed up your private keys, or your seed, and required metadata (such as redeem scripts). You can do it yourself or the help of your friendly techier Bitcoiner friend.

There are, of course, other recovery cases which I didn't cover - such as wallets and coins that don't use these schemes and other types of coins which require special handling.

To wrap it up, I'd want you to feel free to reach out to me ([kobigurk@gmail.com](mailto:kobigurk@gmail.com) or @kobigurk on Twitter) if you have any stuck coins you'd like help with, would like to consult about backing up enough data to be able to recover in the future or wondering about the deeper technical details of how all of this works.
