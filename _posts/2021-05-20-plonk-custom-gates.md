---
title: PLONK custom gates design considerations
author: Kobi
comments: true
---

Thanks to Zac Williamson and Kev for explaining ideas that helped form this document. Thanks to Han for spotting a mistake in "Approach 2" of the MiMC custom gate that could lead to breaking soundness.

# Introduction

PLONK has flexible arithmetization, and can quite easily express complex polynomial gates. "Basic PLONK", as defined in the [paper](https://eprint.iacr.org/2019/953.pdf), defines only fan-in 2 arithmetic gates. The purpose of this document is to show what are custom gates and how they affect the performance of a PLONK proof.

At a high level, PLONK defines the gates as part of a "quotient polynomial" $$t(x)$$ - if this polynomial is zero at the points that represents gates, then the constraint system is satisfied. 

The arithmetic gate contribution to the quotient polynomial is defined as:

$$a(X)b(X)q_M(X) + a(X)q_L(X) + b(X)q_R(X) + c(X)q_O(X) + PI(X) + q_C(X)$$

For example, a multiplication gate that calculates $$xy = z$$ is going to have:

* $$a(g^i) = x$$
* $$b(g^i) = y$$
* $$c(g^i) = z$$
* $$q_M(g^i) = 1$$
* $$q_O(g^i) = -1$$
* $$q_L(g^i), q_R(g^i), PI(g^i), q_C(g^i) = 0$$


# Basic PLONK

"Basic PLONK" has the following performance characteristics:
1. Proof size of $$9\mathbb{G}_1$$ + $$7\mathbb{F}$$ arising from:
    a. Commitments to the wires polynomials $$a(x)$$, $$b(x)$$, $$c(x)$$
    b. Commitment to the permutation polynomial $$z(x)$$
    c. Commitments to 3 $$n-$$degree polynomials, components of the $$3n-$$degree quotient polynomial $$t(x)$$
    d. Commitment to the opening polynomial $$W_z(x)$$, ensuring sure the evaluations of $$t(x), r(x), a(x), b(x), c(x), S_{\sigma_1}, S_{\sigma_2}$$ are correct at $$z$$
    e. Commitment to the opening polynomial $$W_{zw}(x)$$, ensuring the evaluation of $$z(x)$$ is correct at $$zw$$
2. Proving complexity of:
    a. $$9n$$ $$\mathbb{G}_1$$ exponentiations resulting from computing commitments of polynomials of degree $$n$$: 3 from 1a, 1 from 1b, 3 from 1c, 1 from 1d, 1 from 1e
    b. Roughly $$54n$$ field operations from FFTs:
    * 8 of degree $$4n$$ from:
        * The contribution of the permutation argument - 3 for $$\sigma_{0,1,2}
        * 1 for $$z$$
        * 3 for the $$a, b, c$$ wires polynomials
        * 1 inverse transform for the quotient polynomial itself at degree $$4n$$
    * 6 of degree $$2n$$ from:
        * The contribution of the arithmetic gate - $$q_M, q_L, q_R, q_O$$ and $$q_C$$
        * 1 inverse transform for the quotient polynomial itself at degree $$2n$$
    * 12 of degree $$n$$ from:
        * 12 inverse transforms on all the polynomials we need to perform FFT on
3. Verification complexity of (discounting field operations and hashes):
    * 18 exponentiations in $$\mathbb{G}_1$$:
        * 6 from the linearization polynomial opening:
            * 5 from the arithmetic gate polynomials $$q_M, q_L, q_R, q_O$$ and $$q_C$$
            * 1 from the $$z$$ polynomial
        * 9 from the quotient polynomial opening:
            * 3 from the $$n-$$degree parts of $$t$$
            * 3 from the wire polynomials $$a, b$$ and $$c$$
            * 2 from the permutation polynomials $$s_{\sigma1}, s_{\sigma2}$$
        * 1 from the evaluation commitment - containing all the evaluations of the opened polynomials
        * 1 from the polynomial commitment argument for evaluation at $$z$$ - the random opening point
        * 1 from the polynomial commitment argument for evaluation at $$zw$$ - the random opening point + 1
        
Given $$n$$ being the number of arithmetic gates, basic PLONK has a quotient polynomial of degree $$3n$$. The arithmetic gate part itself has only degree $$2n$$, and the $$3n$$ degree stems from the permutation argument on the wire polynomials.

# Designing a custom gate

When designing a custom gate, you can change multiple characteristics.

## Existing wires

If you're introducing a new polynomial gate on existing wires, you affect:
1. Quotient polynomial contribution, which defines the constraints the custom gate verifies
2. Linearization polynomial contribution, which defines a partially evaluated quotient polynomial contribution, such that the contribution doesn't contain polynomial products of committed polynomials

If you increase the degree of the quotient polynomial, you will affect:
1. If the degree of the gate is $$< 3n$$, then you pay in FFTs of degree $$2n$$ on new selector polynomials, and if there's a product of selector polynomials, then opening of some of them until you reach a linear polynomial
2. If the degree of the gate is $$> 3n, \leq 4n$$ (meaning the degree of the quotient polynomial is still $$\leq 3n$$), then you similarly pay in openings and FFTs become of degree $$4n$$
3. If the degree of the gate is $$> 4n$$ and we still want to use the same fast-proving method, you pay in:
    * Similarly, in openings
    * FFTs of degree $$>4n$$. E.g., if degree is $$5n$$ and assuming radix-2 FFTs, then FFT of degree $$8n$$
    * Another commitment for every $$n-$$degree increase, making the proof larger by one group element and an additional scalar multiplication of size $$<=n$$. Additionally, the verifier will pay a scalar multiplication for each $$n-$$degree increase
    
Additionally, you pay a scalar multiplication in verification for each selector polynomial you introduce.

## Adjacent wires

If you're introducing adjacent wires ($$a_{i+1}, b_{i+1}, c_{i+1}$$), you need to add the opening of these values at $$zw$$, so 1 additional field element for each of the required openings, but no significant increase in prover/verifier times - the prover needs to evaluate at the new points, and the verifier already has a polynomial commitment opening at $$zw$$, so it folds into that

## Further wires

If you want access to more wires, e.g. $$a_{i+k}, b_{i+k}, c_{i+k}$$, you similarly needs an opening at $$zw^{k}$$, so 1 additionaly field element for each of the required openings. The prover similarly to the adjacent wires case needs to evaluate at additional points. There is another significant cost increase for both the prover and the verifier though - the prover needs to commit to the opening polynomial at $$zw^{k}$$, which is another $$n-$$degree exponentiation and an additional group element in the proof, and the verifier has to open it, requiring an additional scalar multiplication.

## Additional wires

If you'd like to introduce additional wires above $$a,b,c$$, you could do it in multiple ways.

One way is to introduce wires that are accessible throughout all the gates - introduce more $$n-$$degree polynomials, e.g. $$d(x), e(x)$$ and so on. This would cause:

1. Increase of the permutation polynomial degree, causing the quotient polynomial degree to increase, and in turn increase the FFT degrees, a commitment and a scalar multiplication for each additional wire polynomial because of the $$n-$$degree increase in the quotient polynomial
2. An additional group element in the proof for each additional wire - a commitment to the wire polynomial
3. An additional field element in the proof for each additional wire - the evaluation of the corresponding $$S_{\sigma_i}$$

Another way is to introduce wires that are accessible only for half of the gates, i.e. for the use in exotic gates.

Zac wrote about it:

    The trick is to add a second verification equation for your basic plonk gates.

    Split each degree-n A(X), B(X), C(X) into two degree n/2 polynomials A1(X), A2(X), B1(X), B2(X), C1(X), C2(X).

    Also do this splitting process with the selector polynomials Qm(X), Ql(X), Qr(X), Qo(X), Qc(X).

    In addition to the original plonk gate equation, add a second:

    Qm2(X).A2(X).B2(X) + Ql2(X).A2(X) + Qr2(X).B2(X) + Qo2(X).C2(X) + Qc2(X) + 0 mod Z_H(X)

    You can then add in additional verification equations for exotic gates with 6 wires.

    Because the degrees of your polynomials are half what they used to be, prover compute times haven’t increased (2x the polynomials, but the degrees have been halved). 

    The permutation check now operates on six witness polynomials instead of three - so the maximum degree of your quotient polynomial is still degree 3n.

    This also gives you some extra wiggle room - your exotic 6-wire gate equation can be a degree-6 polynomial, without increasing the degree of the quotient polynomial

    The trade off with all of this, is increased verification time and increased proof size - each new selector polynomial and each new wire commitment will add 1 scalar mul into the verification equations, which starts to add up
    
# Example - MiMC7

To calculate a MiMC7 round function for round $$i$$, we need to calculate $$(x+k+c_i)^7$$, where $$x,k$$ are inputs and $$c_i$$ is a constant

## Approach 1 - using arithmetic gates

We would define the following gates:
1. An add gate to calculate $$t = x+k+c_i$$
2. A multiplication gate to calculate $$t^2$$
3. A multiplication gate to calculate $$t^3$$
4. A multiplication gate to calculate $$t^6$$
5. A multiplication gate to claculate $$t^7$$

In total, 5 constraints.

## Approach 2 - using a custom gate of degree 7

We augment the quotient polynomial with the following contribution:

$$(c(x) - (a(x) + b(x) + q_{mc}(x))^7)q_{mimc}(x)$$

This makes the quotient polynomial of degree $$7n$$.

Assuming we're still using the fast-proving method, this adds 2 scalar multiplications for the verifier, for the two new selector gates. The verifier also pays by 4 additional scalar multiplications because of the 4 degree increase in the quotient polynomial.

The proof becomes larger by 4 group elements.

## Approach 3 - using a custom gate of degree 3 and an adjacent wire

We augment the quotient polynomial with the following contribution:

$$(((c(x) + a(x) + q_{mc}(x))^3 - b(x))\alpha + (c(x) + a(x) + q_{mc}(x)).b(x)^2 - c(xw) )\alpha^2)q_{mimc}(x)$$

This gate works by verifying that $$b(x)$$ contains the $$t^3$$, and proceeds to put $$t^7$$ in $$c_{i+1}$$.

The random $$alpha$$ coefficients enforce that each individual term will be $$0$$.

This doesn't increase the quotient polynomial degree, and thererfore proving times are similar - only a few more evaluations.

We pay in:
1. 2 scalar multiplications by the verifier for the two new selectors $$q_{mc}(x), q_{mimc}(x)$$
2. Additional field element in the proof - $$c(zw)$$
3. Additional field element in the proof - $$q_{mc}(z)$$
4. We don't pay in a field element for $$q_{mimc}(z)$$ - it's reflected in the linearizer

# Conclusion

We've listed a few methods to use the flexibility of the arithmetization step and listed different trade-offs a custom gate designer can take, together with an example of a MiMC7 gate.

We hope that this note proves useful for those learning and using custom gates.
