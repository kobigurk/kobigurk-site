---
title: Additive Homomorphic Encryption on Ethereum
author: Kobi
comments: true
---
A few months ago, the narrative in the cryptocurrency world changed from bitcoin to blockchain. Suddenly, bitcoin is old news and private blockchains are here to change the world. Apparently, blockchain is turning the financial world upside down.

OK, let's be serious - private blockchains are in the exploration phase. Financial institutions, insurance firms and many others are finding out how they can use the technology, or at least some of its concepts. One way to look at them is as a specialized database for specific use-cases. Projects like [HyperLedger](https://www.hyperledger.org/){:target="_blank"} and [Corda](http://r3cev.com/blog/2016/4/4/introducing-r3-corda-a-distributed-ledger-designed-for-financial-services){:target="_blank"} build infrastructures that will power some use-cases. 

But today I'd like to discuss something a bit different. For quite some time, I've been interested in privacy on the blockchain. How can we add some privacy properties to public transactions while still gaining benefit from the underlying blockchain (such as finality, limited auditing, etc)? There are a few interesting ideas, some of them are explored in Vitalik's blog post about [Privacy on the Blockchain](https://blog.ethereum.org/2016/01/15/privacy-on-the-blockchain/){:target="_blank"}.

#So, what can we do?
One approach which I wanted to explore is homomorphic encryption. From [Wikipedia](https://en.wikipedia.org/wiki/Homomorphic_encryption){:target="_blank"}:
> Homomorphic encryption is a form of encryption that allows computations to be carried out on ciphertext, thus generating an encrypted result which, when decrypted, matches the result of operations performed on the plaintext.

For example, assuming \\(ENC\\) and \\(DEC\\) are our encryption and decryption functions respectively, we can have:

$$
x\_1=3, x\_2=4 \\\\ 
c\_1 = ENC(x\_1), c\_2 = ENC(x\_2) \\\\
c\_3 = F(c\_1, c\_2) \\\\
 \text{ such that} \\\\
x\_3 = DEC(c\_3) = x\_1 + x\_2
$$

By performing an operation on the encrypted \\(x\_1, x\_2\\) we got their sum.

The "holy grail" of homomorphic encryption is [Fully Homomorphic Encryption](https://en.wikipedia.org/wiki/Homomorphic_encryption#Fully_homomorphic_encryption){:target="_blank"}, with which you can get results of arbitrary computations on encrypted data, not just their sum or product. One cryptosystem which provides you with *additive* homomorphic encryption is the [Paillier Cryptosystem](https://en.wikipedia.org/wiki/Paillier_cryptosystem){:target="_blank"}. Additive here means that you can get the sum of two encrypted numbers. 

The Paillier Cryptosystem is an asymmetric encryption scheme, meaning that you encrypt with your public key and decrypt with your private key. Leaving the key generation process aside, the parameters we need to use the cryptosystem are:
$$
\\text{public key} = (n, g) \\\\
\\text{private key} = (\lambda, \mu)
$$
The specific meaning of the parameters is not needed for what I want to show, and they're all the result of the key generation process.

**Encryption process**:

1. Take \\(m\\), the number you want to encrypt. \\(m\\) is a member of \\(\mathbb{Z}_n\\).
2. Generate a random number \\(r\\).
3. The encrypted number will be \\(c = g^m \cdot r^n \mod n^2\\).

**Decryption process**:

1. Take \\(c\\), the encrypted number, which is in \\(\mathbb{Z}^*_{n^2}\\).
2. The decrypted number will be \\(m = \frac{(c^{\lambda} \mod n^2) - 1}{n} \cdot \mu \mod n\\).

What we're interested in now is the homomorphic addition property:
$$DEC(ENC(m\_1) \cdot ENC(m\_2)) = m\_1 + m\_2$$

#Homomorphic Encryption in Practice
So what does that property give us? Let's look at a specific use case. Let's assume we have an Ethereum smart contract that manages the expenses of employees: 

1. Every day, employees submit their expenses to the contract. They don't wan the other employees to know their expenses, so they encrypt it and send it to the contract. 
2. The contract adds the encrypted expense to the current total expenses amount.
2. At the end of every month, the company wants to know the total expenses amount, but doesn't want anyone else to know it. They take the encrypted total expenses amount from the contract and decrypt it locally.

This would be a contract with my pre-generated parameters and an initial encrypted total expenses amount of 3:
```javascript
contract paillierBalance {                
    uint nSquared;
    uint public encryptedBalance = 0x2a6a2efecccf5f09e883b87528104e505dedb63fee8de93ccc059116bf32ae5f;
    uint public g = 0xb033814c46b1c673d80ad171adbcf4bc;   
    address public controller;
    uint public n = 0xb033814c46b1c673d80ad171adbcf4bb;
      
    function paillierBalance() {                                      
        controller = msg.sender;
        uint nSquaredTemp;
        uint nInner = n;
        assembly {
            let _n := nInner
            nSquaredTemp := exp(_n, 2)
        }
        nSquared = nSquaredTemp;
    }
                                 
    modifier onlyController {
        if (msg.sender != controller) throw;  
        _
    }

    function homomorphicAdd(uint encryptedChange) onlyController {
        uint encryptedBalanceInner = encryptedBalance;              
        uint nSquaredInner = nSquared;              
        uint encryptedBalanceTemp;                                          
        assembly {
            let _encryptedBalance := encryptedBalanceInner                  
            let _encryptedChange := encryptedChange             
            let _nSquared := nSquaredInner                  
            encryptedBalanceTemp := mulmod(_encryptedBalance, _encryptedChange,_nSquared)
        }
        encryptedBalance = encryptedBalanceTemp;                            
    }
}
```


The encryption and decryption would be client side, using a [python paillier library](https://github.com/kobigurk/paillier){:target="_blank"}. That means that the company first encrypts the initial balance, and whenever the employees want to report the expense, they use the **homomorphicAdd** function. This function uses the homomorphic addition property to store the result of the sum of the previous balance and the new expense.

In this manner, the contract has only one storage variable for the balance while employees can report an unlimited number of expenses. 

#Conclusion
To summarize, this shows a method with which we can preserve privacy while still have some auditing properties. In this example, people can know that they can trust that the final encrypted balance represents the sum of all of the expenses reliably.

What kind of businesses would use methods like this? Time will tell. This is the moment that companies can be creative and find out new ways to architect their systems to work in a more transparent and efficient world. 

All in all, I feel there is a great potential in using this kind of methods and I would love to discuss ideas. Feel free to [contact me](/contact.html)!
