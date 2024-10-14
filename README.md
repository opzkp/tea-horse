# The Tea/Horse Proving System

by Weiji Guo from Lightec Labs

*Note: by the time of writing, this document has NOT been reviewed. It may contain mistakes of various kinds. Also most claims are NOT backed by benchmarks. This is so far a live document rather than a paper draft intended to be published. We will address reviewing and benchmarking as we move on.*

*Note: there are some issues rendering latex in GitHub/Browser (Safari better than Chrome). You may also download the document to preview in VS Code.*

We are developing the Tea/Horse proving system as the underlying scheme for the proposed OP_ZKP soft-fork upgrade to Bitcoin.

### Introduction

The Tea/Horse proving system consists of three major parts:

- Inner-Product Argument (IPA), thus the (EC)DLP hardness assumption only, and the secp256k1 curve only.
- Aggregated Proving to effectively reduce the symptotical Verifier complexity to sub-linear.
- Single circuit for a recursive verifier, to address the linear verification key issue. The recursive verifier verifies zkStark proofs. 

The name Tea/Horse comes from an ancient confidential transaction style, in which people traded tea for horse or vice versa, (perhaps sometimes) keeping the price/bidding information within sleeves, exchanged during hand-shaking. Below picture was taken in Songpan where the author attended a trail running event early July. He came up with the idea of aggregated proving on his flight going there.

<img src="img/th.jpeg" alt="confidential transaction trading tea vs horse" width="300"/>

### Technical Requirements and Key Design

As had been explained in [an email to the bitcoin developer mailing list](https://groups.google.com/g/bitcoindev/c/YEXcac4FMGc), we are considering **Inner-Product Argument**. We could achieve succinct (sub-linear) verifier with **aggregated proving**, but are left with one open issue. That is, the verification key is too large for on-chain deployment.

Searching for a commitment scheme to address this open issue does not land a definitive answer. Instead, it seems quite the opposite, that we might not be able to find such a scheme. Circuit constants are essentially matrixes of size O(N^2), however sparse. And as pointed out before, these matrixes may not be sparse any more after adopting aggregated proving.

Luckily there is a third option besides (1)committing to the circuit constants, or (2)turning to [Dory](https://eprint.iacr.org/2020/1274.pdf). That is, **recursive verification**. The idea is straightforward. Instead of verifying proofs from individual circuits directly, the OP_ZKP implementation has a verifier to verify proofs from one special circuit (the recursive circuit), which in turn recursively verifies proofs for business logic related circuits (application circuits).

The advantages are two fold:

(1) There is no need to deploy verification keys on-chain. The constants (and scheme-related public parameters) of the recursive verifier could be hard-coded in the Bitcoin client, taking up no block space. As for all application circuits, their circuit constants could be provided as witness to the recursive prover, thus hidden from the recursive verifier.

(2) It is possible to upgrade the recursive prover, the recursive verifier or the circuit without interrupting the applications based on OP_ZKP, in case of critical security patches. Still, it is each application circuit's responsibility to take care of their own security and efficiency.

Note that application circuits need not adopt the aggregated proving technique. As the verification is rather different with that of *normal* circuits, it is unlikely to have the recursive verifier support both ways.

### Preliminary

We use bold lower case letters to denote vectors, for example, $\textbf{v} = (v_ 0, v_ 1, v_ 2, ..., v_ {d-1})$, with explicit or implicit length $d$; bold number or its value representation to denote the vector of that number in increasing exponents, for example, $\textbf{2}$ = $(1, 2, 4, 8, ..., 2^{d-1})$ and $\textbf{x}_ \textbf{0}$ = $(1, x_ 0, x_ 0^2, ..., x_ 0^{d-1})$; bold upper case letters to denote matrixes, for example, $\textbf{W}$. 

We use $[m]$ to denote close range $[1, m]$. $[\cdot]\cdot$ denotes scalar multiplication for an implicit EC group $\mathbb{G}$ of order $p$. $\langle \cdot, \cdot \rangle$ denotes the inner product operator, regardless of the types of the parameters: they could be either $\mathbb{Z}_ p \times \mathbb{Z}_ p \rightarrow \mathbb{Z}_ p$, or $\mathbb{Z}_ p \times \mathbb{G} \rightarrow \mathbb{G}$.

Throughout this document, the commitment scheme is Pedersen vector commitment. We also modify the Inner-Product Argument from [Bulletproofs](https://eprint.iacr.org/2017/1066.pdf) a bit following the [Halo paper](https://eprint.iacr.org/2019/1021.pdf), as one of the two vectors are usually known.

#### Pedersen Vector Commitment

For EC group $\mathbb{G}$ of order $p$, let $\textbf{G} \in \mathbb{G}^d, H \in \mathbb{G}$ be random generators. We commit to a vector $\textbf{v} \in \mathbb{Z}_ p^d$ as

$$\text{Commit}(\textbf{G}, H, \textbf{v}, r): C = \langle \textbf{v}, \textbf{G} \rangle + [r]H$$

where $r \in \mathbb{Z}_ p$ is a random blinder providing hiding.

We may ignore the public parameters $\textbf{G}$ and $H$ in the parameter list, and simply put

$$\text{Commit}(\textbf{v}, r): C = \langle \textbf{v}, \textbf{G} \rangle + [r]H \quad \quad \quad (1)$$

$\textbf{G}$ and $H$ should be fixed for Bitcoin use.

To open the commitment $(C, r)$, check if the identity holds

$$\text{Open}(C, \textbf{v}, r): C \overset{?}{=} \langle \textbf{v}, \textbf{G} \rangle + [r]H$$

#### Opening Commitments with Modified IPA

IPA reduces the communication cost of opening a Pedersen vector commitment from linear to $O(log(n))$. We adopt a modified version of IPA based on [Halo](https://eprint.iacr.org/2019/1021.pdf), as in the case of opening a commitment $C$ of a polynomial $p(X)$, the opening point is usually provided by Verifier, thus known.

For polynomial $p(X)$ of degree less than $d$, let $\textbf{v}$ represents the coefficiencies of $p$: 

$$p(X) = \sum_ {i=0}^{d-1}v_ i \cdot X^i $$

Let its commitment value $C$ be defined according to Formula 1. Suppose $p$ evaluates to $a$ at $x_ 0$:

$$a = p(x_ 0) = \sum_ {i=0}^{d-1}v_ i \cdot x_ 0^i = \langle \textbf{v}, \textbf{x}_ \textbf{0}\rangle$$

To open $C$ at $x_ 0$ to value $a$, we use another random generator $U \in \mathbb{G}$, such that:

$$P = C + [a]U = \langle\textbf{v}, \textbf{G} \rangle + [r]H + [\langle \textbf{v}, \textbf{x}_ \textbf{0} \rangle]U \quad \quad \quad (2)$$

Instead of providing $\textbf{v}$ (and $r$) to open the above commitment, we reduce it to another commitment with half the size (assuming $d = 2^k$ for some $k > 0$). To do so, let $\textbf{v}_ {odd}$ and $\textbf{v}_ {even}$ denotes the odd-index part and even-index part of $\textbf{v}$, and similarly $\textbf{G}_ {odd}$ and $\textbf{G}_ {even}$, $\textbf{x}_ {0_ {odd}}$ and $\textbf{x}_ {0_ {even}}$:

$$\textbf{v}_ {odd} = (v_ 1, v_ 3, ..., v_ {d-1}) \quad \textbf{v}_ {even} = (v_ 0, v_ 2, ..., v_ {d-2})$$

$$\textbf{G}_ {odd} = (G_ 1, G_ 3, ..., G_ {d-1}) \quad \textbf{G}_ {even} = (G_ 0, G_ 2, ..., G_ {d-2})$$

$$\textbf{x}_ {0_ {odd}} = (x_ 0, x_ 0^3, ..., x_ 0^{d-1}) \quad \textbf{x}_ {0_ {even}} = (1, x_ 0^2, ..., x_ 0^{d-2})$$

Given a challenge value $\gamma \in \mathbb{Z}_ p$, define new vectors of half the size:

$$\textbf{v}' = \textbf{v}_ {odd} + \gamma \cdot \textbf{v}_ {even}$$

$$\textbf{G}' = \textbf{G}_ {odd} + [\gamma^{-1}]\textbf{G}_ {even}$$

$$\textbf{x}_ \textbf{0}' = \textbf{x}_ {0_ {odd}} + \gamma^{-1} \cdot \textbf{x}_ {0_ {even}}$$

The new commitment value becomes

$$P' = \langle \textbf{v}', \textbf{G}' \rangle + [r]H + [\langle \textbf{v}', \textbf{x}_ \textbf{0}' \rangle]U$$

$$= P + [\gamma](\langle \textbf{v}_ {even}, \textbf{G}_ {odd} \rangle + [\langle \textbf{v}_ {even}, \textbf{x}_ {0_ {odd}} \rangle]U) + [\gamma^{-1}](\langle \textbf{v}_ {odd}, \textbf{G}_ {even} \rangle + [\langle \textbf{v}_ {odd}, \textbf{x}_ {0_ {even}} \rangle]U) $$

$$= P + [\gamma]L + [\gamma^{-1}]R$$

where $L$ and $R$ are provided from Prover to Verifier before the latter generating the challenge value $\gamma$:

$$L = \langle \textbf{v}_ {even}, \textbf{G}_ {odd} \rangle + [\langle \textbf{v}_ {even}, \textbf{x}_ {0_ {odd}} \rangle]U$$

$$R = \langle \textbf{v}_ {odd}, \textbf{G}_ {even} \rangle + [\langle \textbf{v}_ {odd}, \textbf{x}_ {0_ {even}} \rangle]U$$

Based on the values $P, L, R, \gamma$, Verifier could calculate $P'$ on its own.

Rename $(P', \textbf{G}', \textbf{v}', \textbf{x}'_ \textbf{0})$ to $(P, \textbf{G}, \textbf{v}, \textbf{x}_ \textbf{0})$. Repeat this process till $\textbf{v}'$ contains only one element, than Prover can directly provide its value to Verifier. Verifier checks if the commitment opens correctly.

*Remark* Chosing odd/even rather than lower/higher halves is based on the need to align many sub-circuits with different size.

#### Batched Opening

##### Same Polynomial, Multiple Points
First we describe how to open a commitment value $C$ to a polynomial $p(X)$ at multiple points $(x_ 1, x_ 2, ..., x_ m)$ with IPA. Suppose we have $a_ i = p(x_ i), i \in [m]$, we need to open $C$ at each $x_ i$ to $a_ i$. 

Let $U_ i, i \in [m]$ be $m$ random generators. At the begining we shall have 

$$P = C + \sum_ {i=1}^m[{a_ i}]U_ i = \langle\textbf{v}, \textbf{G} \rangle + [r]H + \sum_ {i=1}^m[\langle \textbf{v}, \textbf{x}_ \textbf{i} \rangle]U_ i \quad \quad \quad (3)$$

For each round, in a similar way we may have:
$$\textbf{x}_ \textbf{i}' = \textbf{x}_ {i_ {odd}} + \gamma^{-1} \cdot \textbf{x}_ {i_ {even}}$$

and

$$P' = P + [\gamma]L + [\gamma^{-1}]R$$

$$L = \langle \textbf{v}_ {even}, \textbf{G}_ {odd} \rangle + \sum_ {i=1}^m[\langle \textbf{v}_ {even}, \textbf{x}_ {i_ {odd}} \rangle]U_ i$$

$$R = \langle \textbf{v}_ {odd}, \textbf{G}_ {even} \rangle + \sum_ {i=1}^m[\langle \textbf{v}_ {odd}, \textbf{x}_ {i_ {even}} \rangle]U_ i$$

Besides necessary values of $a_ i$, there are no extra communication costs.

##### Different Polynomial, Same Point

Next we describe how to open multiple polynomials at the same point. Suppose we have $m$ polynomials $p_ i(X)$ for $i \in [m]$, each is evaluated to $a_ i$ at the same point $x_ 0$:

$$a_ i = p_ i(x_ 0), i \in [m]$$

Prover first need to commit to each of the polynomials. Let the commitments be $C_ i, i \in [m]$. For simplicity, let's use $\textbf{p}_ i$ instead of $\textbf{v}$ to represent the the vector of coefficiencies of polynomial $p_ i(X)$. We have:

$$C_ i = \text{Commit}(\textbf{p}_ i, r_ i) = \langle \textbf{p}_ i, \textbf{G} \rangle + [r_ i]H$$

Verifier generates a random value as challenge: $\beta \in \mathbb{Z}_ p$. Prover computes:

$$C = \sum_ {i=1}^m \beta^i \cdot C_ i = \sum_ {i=1}^m \langle \beta^i \cdot \textbf{p}_ i, \textbf{G} \rangle + [\beta^i \cdot r_ i]H \quad \quad \quad (4)$$

Equivalently there exists a batched polynomial

$$p(X) = \sum_ {i=1}^m \beta^i \cdot p_ i(X)$$

which is commited to $C$, with $r = \sum_ {i=1}^m \beta^i \cdot r_ i$ as blinder:

$$C = \text{Commit}(\textbf{p}, r)$$

And this commitment value $C$ shall be open to $a = \sum_ {i=1}^m \beta^i \cdot a_ i$ at $x_ 0$. For that we already have a protocol.

##### Hybrid Case

We might have hybrid case for the Tea/Horse proving system: opening $R$ for $r(X, 1)$ at two different points $z$ and $yz$, and opening $T_ {lo}$ for $t_ {lo}(X, y)$ and $T_ {hi}$ for $t_ {hi}(X, y)$ at $z$. A combination of the previous two cases suffices here. We leave the details to later sections.

Note that the communication costs here are three $\mathbb{G}$ elements for commitments and four $\mathbb{Z}_ p$ elements for open-to values, in addition to what is required by IPA. The verifier complexity is still linear to the polynomial size.

### The Proving System

In the Tea/Horse proving system, a typical R1CS circuit is organized as many (m) sub-circuits, each having a size of fixed upbound set to $2^{16}$. In this section, we describe how the system works.

The basic idea is to aggregatedly prove all the sub-circuits. For each sub-circuit, we follow a protocol which is a mixture of Sonic and Halo. Then we have an argument to aggregate them together.

#### Sonic Arithemtic

In this section we reiterate the Sonic Arithmetic following notations from [Sonic Paper](https://eprint.iacr.org/2019/099.pdf) and [Halo paper](https://eprint.iacr.org/2019/1021.pdf) with slight modifications introducing matrix notations. We assume that the circuit has been preprocessed (see Appendix A of [Bootleproofs](https://eprint.iacr.org/2016/263.pdf)).

Suppose we have an arithmetic circuit $C$ and let $N$, $Q$, $d$, $k$ be integers such that $d = 4N = 2^k$ and $3Q \lt d$. There are $N$ multiplication gates such that the $\textit{i}$-th constaint is:

$$a_ i \cdot b_ i = c_ i \quad \quad \quad (5)$$

where $a_ i, b_ i, c_ i$ are the $i$-th element of the witnesses vectors $\textbf{a}, \textbf{b}, \textbf{c} \in \mathbb{Z}_ p^{N}$.

For the $Q$ linear constraints capturing the copy wires, multiplied-by-constant gates and addition gates, we denote $\textbf{U}, \textbf{V}, \textbf{W} \in \mathbb{Z}_ p^{Q \times N}$ and $\textbf{k} \in \mathbb{Z}_ p^Q$ such that:

$$\textbf{U}\cdot\textbf{a} + \textbf{V}\cdot\textbf{b} + \textbf{W}\cdot\textbf{c} = \textbf{k} \quad \quad \quad (6)$$

Here, $\textbf{U}, \textbf{V}, \textbf{W}, \textbf{k}$ are the circuit constants (where there is no confusion, we use row vector and column vector interchangedly).

For $\textbf{M} \in \{\textbf{U}, \textbf{V}, \textbf{W}\}$, we denote 

$$m_ i(Y) = \sum_ {q=1}^Q Y^q\cdot M_ {q,i} $$

where $M_ {q,i}$ denotes the element at the position of $(q, i)$ of matrix $\textbf{M}$. 
And let 

$$k(Y) = \sum_ {q=1}^Q Y^q \cdot k_ q$$

Embedding all the constraints into a single equation of $Y$, we have

$$ Y^N (\sum_ {i=1}^N (a_ i u_ i(Y) + b_ i v_ i(Y) + c_ i w_ i(Y)) - k(Y)) + \sum_ {i=1}^N (a_ i b_ i - c_ i)\cdot (Y^i + Y^{-i}) = 0 \quad \quad \quad (7)$$

which should hold at all points if all the constraint system is satisfied for some witness $\textbf{a, b, c} \in \mathbb{Z}_ p^N$. Given a large enough field, this means the constraint system is satisfied with high probability if Equation 7 holds for a random challenge value $y \in \mathbb{Z}_ p$ and some witness values.

Going on with Sonic Arithmetic, let's define some more polynomials:

$$r(X, Y) = \sum_ {i=1}^N(a_ iX^iY^i + b_ iX^{-i}Y^{-i} + c_ iX^{-i-N}Y^{-i-N})$$

$$s(X, Y) = \sum_ {i=1}^N(u_ i(Y)X^{-i} + v_ i(Y)X^i + w_ i(Y)X^{i+N})$$

$$s'(X, Y) = Y^Ns(X, Y) - \sum_ {i=1}^N(Y^i + Y^{-i})X^{i+N}$$

$$t(X, Y) = r(X, 1)(r(X, Y) + s'(X, Y)) - Y^Nk(Y) \quad \quad \quad (8)$$

Note that the constant term of $t(X, Y)$ in terms of $X$, that is, the coefficient of $X^0$, is exactly the left-hand side of Equation 7. So the protocol design is to show that, Equation 8 still holds after removing the constant term from the left-hand side.

#### Single Sub-Circuit Proving

This part is still mostly Sonic Arithmetization with (modified) IPA. Some ideas come from [Halo](https://eprint.iacr.org/2019/1021.pdf) but we do not use the amortizing part or cycles of curves part. We stick to the `secp256k1` curve. Also as we hard-coded some circuit constants in the verifier side, verifier could compute these values on its own. 

Prover commits to $r(X, 1)$ with blinder $\delta_ R$ for circuit $C$. We follow [Halo paper](https://eprint.iacr.org/2019/1021.pdf) to scale it with $X^{3N-1}$ to ensure $r(X, 1)$ is at most degreen $N$.

$$\delta_ R \overset{\$}{\leftarrow} \mathbb{Z}_ p,\quad R \leftarrow \text{Commit}(r(X, 1)X^{3N-1}; \delta_ R) $$

Verifier responds with a challenge value $y \in \mathbb{Z}_ p$. In a non-interactive setting, $y$ should be generated by hashing the past transcriptions up to and including the commitment value $R$. We ignore this notion in later sections.

Now let $t_ {lo}(X, y), t_ 0(y), t_ {hi}(X, y)$ be the part of $t(X, y)$ with negative exponent, constant term, and with positive exponent, for X, that is,

$$t(X, y) = t_ {lo}(X, y)X^{-d} + t_ 0(y) + t_ {hi}(X, y)X$$

Prover commits to $t_ {lo}$ and $t_ {hi}$.

$$\delta_ {T_ {lo}} \overset{\$}{\leftarrow} \mathbb{Z}_ p,\quad T_ {lo} \leftarrow \text{Commit}(t_ {lo}(X, y); \delta_ {T_ {lo}}) $$

$$\delta_ {T_ {hi}} \overset{\$}{\leftarrow} \mathbb{Z}_ p,\quad T_ {hi} \leftarrow \text{Commit}(t_ {hi}(X, y); \delta_ {T_ {hi}}) $$

Verifier responds with a challenge value $z \in \mathbb{Z}_ p$.

Prover now needs to open a few commitments, and provides open to values and proofs.

$$(e = r(z, 1), proof_ a) \leftarrow \text{Open}_ \text{IPA}(R, z, r(X, 1), \delta_ R)$$

$$(f = r(z, y), proof_ b) \leftarrow \text{Open}_ \text{IPA}(R, yz, r(X, 1), \delta_ R)$$

$$(t_ 1 = t_ {lo}(z, y), proof_ {t_ {lo}}) \leftarrow \text{Open}_ \text{IPA}(T_ {lo}, z, t_ {lo}(X, y), \delta_ {T_ {lo}})$$

$$(t_ 2 = t_ {hi}(z, y), proof_ {t_ {hi}}) \leftarrow \text{Open}_ \text{IPA}(T_ {hi}, z, t_ {hi}(X, y), \delta_ {T_ {hi}})$$

Verifier need to verify all the above openings, compute $s = s(z, y)$, $s'$ from $s$, $k = k(y)$, then check if

$$t_ 1 + t_ 2 \overset{?}{=} e(f + s') - k \quad \quad \quad (9)$$

Passing Identity Test 9 means that $t_ 0(y) = 0$ with overwhelming probability. If all checks pass, Verifier returns 1, otherwise 0.

*Remark* The computation of $s$ does not involve confidential witness data $\textbf{a}, \textbf{b}$ or $\textbf{c}$, so Verifier could compute on its own. Opening a commitment to $s(X, y)$ is not viable for aggregated proving, as direct computation in $\mathbb{Z}_ p$ should be much faster than verifying opening with IPA. Also the size of none-zero entries in the aggregated $\textbf{U}, \textbf{V}$ or $\textbf{W}$ might exceed $d$. That's the reason we don't bother committing to it even for single sub-circuit. We will discuss the optimization for computing $s$ in later sections.

##### Batched Opening for Sub-Circuit

The opening of $R$ at $z$, $yz$, of $T_ {lo}$ and $T_ {hi}$ at $z$, could be batched together with the batching techniques we have described in the last section.

Specfically, $R$ could be opened to $e \in \mathbb{Z}_ p$ and $f \in \mathbb{Z}_ p$ at $z$ and $yz$ given random generators $U_ 1$ and $U_ 2$. We have

$$P_ R = R + [e]U_ 1 + [f]U_ 2 = \langle\textbf{r}, \textbf{G} \rangle + [\delta_ R]H + [\langle \textbf{r}, \textbf{z} \rangle]U_ 1 + [\langle \textbf{r}, \textbf{zy} \rangle]U_ 2$$

where $\textbf{r}$ denotes the coefficiencies for $r(X, 1)$.

To open $T_ {lo}$ and $T_ {hi}$ at $z$ in a batch along with $R$, given $\beta$ as the challenge value, we should have

$$R + [\beta]T_ {lo} + [\beta^2]T_ {hi} + [e + \beta \cdot t_ 1 + \beta^2 \cdot t_ 2]U_ 1 + [f]U_ 2$$

$$\quad = \langle\textbf{r} + \beta \cdot \textbf{t}_ \textbf{lo} + \beta^2 \cdot \textbf{t}_ \textbf{hi}, \textbf{G} \rangle + [\delta_ R + \beta \cdot \delta_ {lo} + \beta^2 \cdot \delta_ {hi}]H $$

$$\quad \quad + [\langle \textbf{r} + \beta \cdot \textbf{t}_ \textbf{lo} + \beta^2 \cdot \textbf{t}_ \textbf{hi}, \textbf{z} \rangle]U_ 1 + [\langle \textbf{r}, \textbf{zy} \rangle]U_ 2 \quad \quad \quad (10)$$

#### Aggregated Proving

We now consider how we could aggregate the proving of $m$ circuits together, denoted $C^{(i)}$ for $i \in [m]$. We use superscript $(i)$ to indicate the $i$-th circuit. We have $N$ multiplication constraints for sub-circuit $C^{(i)}$

$$\textbf{a}^{(i)} \circ \textbf{b}^{(i)} = \textbf{c}^{(i)}$$

where $\circ$ denotes the element-wise multiplication, or Hadamard product, for witnesses $\textbf{a}^{(i)}, \textbf{b}^{(i)}, \textbf{c}^{(i)} \in \mathbb{Z}_ p^N$.

And $Q$ linear constraints:

$$\textbf{U}^{(i)} \cdot \textbf{a}^{(i)} + \textbf{V}^{(i)} \cdot \textbf{b}^{(i)} + \textbf{W}^{(i)} \cdot \textbf{c}^{(i)} = \textbf{k}^{(i)}$$

where $\textbf{U}^{(i)}, \textbf{V}^{(i)}, \textbf{W}^{(i)} \in \mathbb{Z}_ p^{Q \times N}, \textbf{k}^{(i)} \in \mathbb{Z}_ p^Q$ are constants for $C^{(i)}$.

Finally we can define polynomials $r^{(i)}(X, Y)$, $s^{(i)}(X, Y)$ and $t^{(i)}(X, Y)$ for circuit $C^{(i)}$. Before aggregation could happen, Prover needs to commit to each $r^{(i)}(X, 1)$: 

$$\delta_ R^{(i)} \overset{\$}{\leftarrow} \mathbb{Z}_ p,\quad R^{(i)} \leftarrow \text{Commit}(r^{(i)}(X, 1)X^{3N-1}; \delta_ R^{(i)}) $$

and Verifier responds with a challenge value $y \in \mathbb{Z}_ p$. Note that all sub-circuits share the same $y$ value.

Next, Prover may commit to each of $t_ {lo}^{(i)}(X, y)$ and $t_ {hi}^{(i)}(X, y)$:

$$\delta_ {T_ {lo}^{(i)}} \overset{\$}{\leftarrow} \mathbb{Z}_ p,\quad T_ {lo}^{(i)} \leftarrow \text{Commit}(t_ {lo}^{(i)}(X, y); \delta_ {T_ {lo}^{(i)}}) $$

$$\delta_ {T_ {hi}^{(i)}} \overset{\$}{\leftarrow} \mathbb{Z}_ p,\quad T_ {hi}^{(i)} \leftarrow \text{Commit}(t_ {hi}^{(i)}(X, y); \delta_ {T_ {hi}^{(i)}}) $$

Verifier responds with another challenge value $z \in \mathbb{Z}_ p$, which also holds the same for all sub-circuits. And further two more challenge value $\alpha, \beta \in \mathbb{Z}_ p$. We use $\alpha$ to combine polynomials, commitment values, blinders and open-to values together:

$$r(X, 1) = \sum_ {i=1}^m \alpha^i \cdot r^{(i)}(X, 1) \quad \quad s(X, y)= \sum_ {i=1}^m \alpha^i \cdot s^{(i)}(X, y) $$

$$t_ {lo}(X, y)= \sum_ {i=1}^m \alpha^i \cdot t_ {lo}^{(i)}(X, y) \quad \quad t_ {hi}(X, y)= \sum_ {i=1}^m \alpha^i \cdot t_ {hi}^{(i)}(X, y) $$

$$R = \sum_ {i=1}^m \alpha^i \cdot R^{(i)} \quad \quad T_ {lo} = \sum_ {i=1}^m \alpha^i \cdot T_ {lo}^{(i)} \quad \quad T_ {hi} = \sum_ {i=1}^m \alpha^i \cdot T_ {hi}^{(i)}$$

$$\delta_ R = \sum_ {i=1}^m \alpha^i \cdot \delta_ {R^{(i)}} \quad \quad \delta_ {T_ {lo}} = \sum_ {i=1}^m \alpha^i \cdot \delta_ {T_ {lo}^{(i)}} \quad \quad \delta_ {T_ {hi}} = \sum_ {i=1}^m \alpha^i \cdot \delta_ {T_ {hi}^{(i)}}$$

$$e = \sum_ {i=1}^m\alpha^i\cdot e^{(i)}\quad f = \sum_ {i=1}^m\alpha^i\cdot f^{(i)}\quad t_ 1 = \sum_ {i=1}^m\alpha^i\cdot t_ 1^{(i)}\quad t_ 2 = \sum_ {i=1}^m\alpha^i\cdot t_ 2^{(i)}$$

Then we use $\beta$ to batch open the combined polynomials, applying the Equation 10 with the definitions given above.

After checking that Equation 10 holds, Verifier compute $s, s'$ and $k$, and check if the Identity Test 9 passes.

#### Costs Analysis

For $m$ sub-ircuits each with $N$ multiplication constraints, the proof data include:

- commitments to $r^{(i)}(X, 1), t_ {lo}^{(i)}(X, y), t_ {hi}^{(i)}(X, y)$, totally $3m$ elements of $\mathbb{G}$, denoted $3m[G]$;
- aggregated opening hints $\delta_ R, \delta_ {T_ {lo}}, \delta_ {T_ {hi}}$, totally 3 elements of $\mathbb{Z}_ p$, denoted $3[F]$;
- aggregated open-to values $e, f, t_ 1, t_ 2$, totally 4 elements of $\mathbb{Z}_ p$, denoted $4[F]$;
- proof data for opening the aggregated commitment, which is $2log _ 2(d)[G] + 1[F] = (2log _ 2(N) + 4)[G] + 1[F]$ (remember $d = 4N$). 

And the verification key include:

- elements of combined $\textbf{U}, \textbf{V}, \textbf{W}$, of apparent size $3QN$, which could be further optimized;
- elements of combined $\textbf{k}$, of size $Q$.

Verifier complexity is:

- combining $3m$ commitments to 3 commitments, that is $3$ multi-scalar multiplications, echo of size $m$;
- batch opening the combined commitment, dominated by $d = 4N$ sized multi-scalar multiplication;
- computing the aggregated $s(X, Y)$ and $k(Y)$, of apparent $O(N^2)$ field operations in $\mathbb{Z}_ p$, which could be further optimized.

#### Optimization

##### Data Saving

First observe that combined $\textbf{U}, \textbf{V}, \textbf{W}$ are roughly triangle matrixes, with the upper right half mostly zero. This is due to the fact that linear constraints can only reference existing wires. This cuts half the storage cost and runtime for the verifier.

Next observation is that some circuits are dominated by a few sub-circuits repeated many times. For example, based on the cost analysis above, the recursive verifier for a Tea/Horse proof is dominated by multi-scalar multiplicatioins and some field operations. In that case, the same set of $\{\textbf{U}_ 0, \textbf{V}_ 0, \textbf{W}_ 0\}$ could be used for many sets of data of the same sub-circuit. So the aggregated $\textbf{U}, \textbf{V}, \textbf{W}$ could still be sparse. We conjecture that the actual cost could be more likely $O(N)$ rather than $O(N^2)$, although the constant factor might be concretely large, say, a few tens or hundred.

##### Further Optimizaiton - First Attempt

Last we dicsuss the feasibility of proving the existence of the $3m$ commitments for $r^{(i)}(X, 1), t_ {lo}^{(i)}(X, y), t_ {hi}^{(i)}(X, y)$, with a dedicated circuit. This is the idea borrowed from [Nova](https://eprint.iacr.org/2021/370.pdf), which employs zkSnark to prove the folding process.

*Remark* Note that Nova in itself cannot be applied here as it requires cycles of elliptic curves. Also Nova requires auxillary zkSnark circuits, which has similar performance characteristics as analyzed below.

For commitment values $R^{(i)}, T_ {lo}^{(i)}, T_ {hi}^{(i)}, i \in [m]$, we need a circuit to prove the following computation:

- derive a value $\alpha_ C \in \mathbb{Z}_ p^*$ by hashing $C^{(i)}$ together for $C \in \{R, T_ {lo}, T_ {hi}\}$. A field friendly hash would be helpful. However we might need just `Sha256` for Bitcoin.
- compute $\alpha = \alpha_ R \cdot \alpha_ {T_ {lo}} \cdot \alpha_ {T_ {hi}}$
- use the derived $\alpha$ to combined the commitment values $C = \sum_ {i=1}^m\alpha^iC^{(i)}$ for $C \in \{R, T_ {lo}, T_ {hi}\}$

Verifier verifies the proof of the above computation, instead of computing $R, T_ {lo}, T_ {hi}$ on his own. The $3m$ commitment values $R^{(i)}, T_ {lo}^{(i)}, T_ {hi}^{(i)}$ are replaced with the proof data of the above circuit. So how large would this circuit be, and the proof data?

The circuit needs to 

- hash $3m$ elements of $\mathbb{G}$
- run $3$ multi-scalar multiplication, each of size $m$

Unfortunately, it costs much more to run MSM in-circiut than directly, both proving and verification with IPA.

##### Second Attempt

Take a step back. We could start from $r^{(i)}(X, 1), t_ {lo}^{(i)}(X, y), t_ {hi}^{(i)}(X, y)$ directly and prove correct derivation of $\alpha$ and aggregation of $r(X, 1), t_ {lo}(X, y), t_ {hi}(X, y)$, etc. For $m$ sub-circuits each of size $O(d)$, totally there are $O(dm)$ elements of $\mathbb{Z}_ p$ to process. The costs with IPA are $O(log_ 2(d) + log_ 2(m))$ for proof data, $O(dm)$ size MSM for verification runtime, which is much more expensive than direct computation.

##### Review and Alternatives

If we set $N$ to $2^{16}$, then $m$ might range from a few donzens to a few thousands for reasonable circuits. Under this assumption, Tea/Horse proving system has shorter proof yet longer runtime than [Hyrax](https://eprint.iacr.org/2017/1132.pdf), which typically features $O(\sqrt{n})$ proof size and verifier complexity.

At this point our major issues are (1) with the verification key size, which is $O(N)$ with a big constant factor under our conjecture; and (2), the size of the recursive verifier, which could be very large, given it has to compute a large MSM and many field operations proportional to verification key size.

The application circuits have to be in a scheme with succinct verifier, for example Tea/Horse or Hyrax. But even with Hyrax the recursive verifier is still very large.

###### Sparse Polynomial Commitment

Now let's take a step back and review if we can apply sparse polynomial commitment technique, called `Spark`, which was introduced in [Spartan](https://eprint.iacr.org/2019/550.pdf) and refined in [Lasso](https://people.cs.georgetown.edu/jthaler/Lasso-paper.pdf).

Specifically, we seek to remove the circuit constants from the verifier key, replace them with some commitments. Then we don't need a recursive verifier, the proposed OP_ZKP implementation could verify proof data of application circuits directly. This addresses the two issues mentioned above.

*Remark* This is a live document. We might change our mind or switch to a better solution in the middle of developing the system.

Based on the Figure 6 of the [Spartan paper](https://eprint.iacr.org/2019/550.pdf), we could commit to the constants of each sub-circuit. The circuit constants are of size $O(N)$, then with Hyrax polynomial commitment, `Spark` achieves $O(log^2(N))$ proof size and $O(\sqrt{N})$ verifier time.

Since we are already dealing with $O(N)$ verifier time, $O(\sqrt{N})$ is totally acceptable even with a large constant. It is the proof size that concerns us. $O(log^2(N))$ is too big. 

#### zkStark as the scheme for Application Circuits

We now get back to the recursive verifier design. This time we consider any scheme that has transparent setup, and has no pairing requirement, yet with minimal verifier time so that the recursive verifier could be small. It appears zkStark fits our requirement with its $O(log^2(N))$ verifier time. Also its verifier is dominated by hashing instead of EC scalar-multiplication. This could also help reduce the size of the recursive verifier especially if we adopt some ZK-friendly hash algorithm.

A rough estimation to the verifier cost is based on the assumption that: (1) there exists a hash function that costs only 100-th of `SHA256` in terms of R1CS constraints; (2) hashing computation takes up at least 80% of the verifier computation. Further, let's use blowup factor of 32, and security level of $\lambda = 128$, and the upper bound of sub-circuit size is $N = 2^{16}$.

For an application circuit with $2^{20}$ constraints, there should be $O(ceil(\lambda / log_2(32)) * log_2^2(2^{20})) = 26 \times 400 = 10400$ hash values in the proof. Then to verify the proof in the Tea/Horse system, roughly same amount of hashing operations must be performed in R1CS. 

According to the Table 1 of [Reinforced Concrete](https://eprint.iacr.org/2021/1038.pdf), `Rescue-Prime` has less than $2^8$ constraints to hash 64 bytes of data, which happen to be the input in a Merkle Proof verification. With $N = 2^{16}$ we may pack 256 innovations of `Rescue-Prime` into one sub-circuit. So we need $m' = ceil(10400/256) = 41$ sub-circuits for the hash evaluations for the recursive verifier. In total we need $m = ceil(m'/80\%) = 52$ sub-circuits.

According to costs analysis, the total proof size to be verified by `OP_ZKP` should be $(3m + 2log_2(N) + 4)[G] + 8[F] = 188[G] + 8[F]$. Since elements of $\mathbb{G}$ and $\mathbb{F}$ are of size 32 bytes, therefore the total proof size is 6272 bytes. Note that this is much better than Hyrax's square root proof size, which has at least thousands of group elements in the proof; at the cost of longer runtime of course, which we analyze below.

Verifier runtime includes three parts:

- 3 MSM each with size $m = 52$
- MSM with size $d= 4N = 2^{18}$
- computing $s(X, Y)$ and $k(Y)$

The first one is relatively easy to compute. The seond one might cost single thread of a modern computer around 1 second, and of Raspeberry Pi 4 around 10 seconds.

For the last one, note that $m' = 41$ sub-circuits have exactly the same structure (circuit constants), so we could estimate $s(X, Y)$ to have size around $3 \times 12 \times N = 36N$, or about 2.4 million, field operations, not discounting for potential duplicated entries. Since the field operations are dominated by mulltiplication, we use related benchmarking data to estimate the runtime. Based on the Table 2 of [Speed Optimizations in Bitcoin Key Recovery Attacks](https://eprint.iacr.org/2016/103.pdf), each multiplication costs 0.049us for a laptop computer with the `secp256k1` library. Then 2.4 million multiplication should cost about 0.12 second. (Acutal cost might be a bit higher as we oversimplify here, $s(X, Y)$ is combined from $m$ many sub-circuits and the combination factor is dependent on the witness.)

Overall, verifier cost is dominated by a large MSM, and the total time looks acceptable, even for a small miner (RP4). Still, there is room for fine tuning and optimization. For example, we could have smaller $N$ and larger $m$ at the cost of bigger proof. Cutting $N$ in half leads to half the verifier time yet twice larger the proof. We leave further discussion to later sections.

The conclusion is that, we can go ahead with aggrepated IPA proving for the recursive verifier + zkStark for the application circuits.

#### Batched Verification

We now move to a new topic regarding how a miner could batch verify multiple `OP_ZKP` transactions.

Suppose there are $M$ such transactions, each denoted as $T _I$ for $I \in [M]$. For $T _I$, its proof has $m_I$ sub-circuits.

First the miner need to derive $y _I, z _I, \alpha _I, \beta _I$, retrieve and combine $R^{(i)}, T _{lo}^{(i)}, T _{hi}^{(i)}, i \in [m_I]$ into $R _I, T _{{lo} _I}, T _{{hi} _I}$, retrieve $e _I, f _I, t _{1 _I}, t _{2 _I}, \delta _{R _I}, \delta _{T _{lo _I}}, \delta _{T _{hi _I}}$ for $Proof _I$ of each $T _I$. These values should satisfy Formula (10) as attested by $Proof _I$.

To verify $M$ proofs: $Proof _I$ for $I \in [M]$ together, it suffices to randomly combine all the proofs along with the above values. To facilitate pre-computation and computation reuse, the random value should be derived for each $Proof _I$ individually, i.e., 

$$\gamma _I = \text{hash}(Proof _I)$$

Since all the proofs are created under the same public parameters, they are expected to have the same length $d$. Then we can combine them into one pseudo-proof:

$$Proof = \sum _{I=1}^{M}\gamma _I \cdot Proof _I$$

The resulting value $Proof$ should attest to the combined left hand side of (10):

$$LHS = \sum _ {I=1}^M \gamma _ I \cdot (R _ I + [\beta _ I]T _ {{lo} _ I} + [{\beta _ I}^2]T _ {{hi} _ I} + [e _ I + \beta _ I \cdot t _ {1 _ I} + {\beta _ I}^2 \cdot t _ {2 _ I}]U _ 1 + [f_ I]U _ 2)$$

When we do so, each $\mathbb{G}$ element of $Proof _I$ is scalar-multiplied by $\gamma _I$, then added to elements in the same position.

Note that $Proof$ is still of size $d = 4N$. So basically we reduce $M$ large MSM computation to only one, with the cost of mainly some combination operation, which themselves are scalar-multiplications. We leave the cost analysis to the next sub-section.

After the proof verification, the miner has yet to compute $s, s', k$, then test if (9) holds. As $s _I, s' _I, k _I$ are all dependent on value of $y _I, z _I$, it has to be computed one by one before aggregation:

$$s _ I = \sum_ {i=1}^N(u_ i(y _ I){z _ I}^{-i} + v_ i(y _ I){z _ I}^i + w_ i(y _I){z _I}^{i+N})$$

$$s' _I = {y _I}^N \cdot s _I - \sum _{i=1}^N({y _I}^i + {y _I}^{-i}){z _I}^{i+N}$$

$$s' = \sum _{I=1}^M \gamma _I \cdot s' _I \quad \quad k = \sum _{I=1}^M \gamma _I \cdot k(y _I)$$

##### Cost Analysis and Optimization

Assuming each `OP_ZKP` transaction to be of size less than 7KB, counting in non-proof data such as UTXOs and payment amounts etc. If a bitcoin block consists of `OP_ZKP` transaction only, the maximal value of $M$ is 585.

Combining the proofs together involves $O(2\text{log} _2(d))$, or a few dozens of MSM operations, each with size of $M$. The combined total is still trivial compared to the large MSM of size $d = 2^{18}$.

Each $s _I$ computation costs 0.12s according to earlier estimation, then we shall spend a total of $0.12s \times M = 70s$ to compute $s$ if no optimization is involved. Computing $s'$ from $s$, or $k$ is much faster therefore ignored here. 

This huge computation burden seems inherent to IPA and hard to optimize or remove. Even if it was feasible to optimize, the proof size of `OP_ZKP` transactions will still hinder its adoption. To address both issues together, we consider a new dedicated circuit to further verifies proofs of multiple `OP_ZKP` transactioins together.

#### Application Attestation

With a recursive verifier in play, each Bitcoin script containing the `OP_ZKP` opcode must have the capability to tell if the verified proof is dedicated to it. This is called attestation. The recursive verifier will also attach to its proof a piece of information as the attestation so that the script or the `OP_ZKP` implementation could verify.

### The Block Prover

Suppose there exists a threshold $T$ such that it is cheaper in terms of block space (and verification time) to verify $t \ge T$ `OP_ZKP` transactions in a dedicated circuit, whose proof will be verified by each node instead. Unlike earlier optimization attempts, this time we may consider those ZKP schemes without batched verification features.

With new choices open to us, we might consider zkStark as it provides $O(log _2 ^2(N))$ proof size and verification time, and aggregated proving as well.

Suppose there is the said threshold $T$. Each bitcoin block will be limited to have at most $T$ `OP_ZKP` transactions. Then a regular computer should be able to verify such a block about $0.12s*T + 1$ seconds, without involing multi-threading. In a setting with $T = 50$, the cost is 7 seconds. It is translated to more than 1 minute for a Raspberry Pi 4 miner. But we can adopt multi-threading in such a case to effectively reduce the time requirement. Meanwhile, the total block space occupied by the proof data of these $T$ transaction roughly equal to the replacement proof (subject to a few security parameters and circuit size of the recursive verifier).

Surpassing the threshold, the block prover may kickin to replace the proof data of those $t$ `OP_ZKP` transactions with another proof, which has a size of a few hundred KBs, and the verification time should be very short, probably well within  1 second as most of the computation is to compute hash values.

This documents has been packed with lots of speculated numbers. So we go no further estimating the constraint count, or incentives designed for the block prover. 