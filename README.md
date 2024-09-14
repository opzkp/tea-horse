# The Tea/Horse Proving System

by Weiji Guo from Lightec Labs

*Note: by the time of writing, this document has NOT been reviewed. It may contain mistakes of various kinds. Also most claims are NOT backed by benchmarks. This is so far a live document rather than a paper draft intended to be published. We will address reviewing and benchmarking as we move on.*

*Note: there are some issues rendering latex in GitHub/Browser. You may download the document to preview in VS Code. We will also provide a pdf version later.*

We are developing the Tea/Horse proving system as the underlying scheme for the proposed OP_ZKP soft-fork upgrade to Bitcoin.

### Introduction

The Tea/Horse proving system consists of three major parts:

- Inner-Product Argument (IPA), thus the (EC)DLP hardness assumption only, and the secp256k1 curve only.
- Aggregated Proving to effectively reduce the symptotical Verifier complexity to sub-linear.
- Single circuit for a recursive verifier, to address the linear verification key issue.

The name Tea/Horse comes from an ancient confidential transaction style, in which people traded tea for horse or vice versa, (perhaps sometimes) keeping the price/bidding information within sleeves, exchanged during hand-shaking. Below picture was taken in Songpan where the author attended a trail running event early July. He came up with the idea of aggregated proving on his flight going there.

<img src="img/th.jpeg" alt="confidential transaction trading tea vs horse" width="300"/>

### Technical Requirements and Key Design

As had been explained in [an email to the bitcoin developer mailing list](https://groups.google.com/g/bitcoindev/c/YEXcac4FMGc), we are considering **Inner-Product Argument**. We could achieve succinct (sub-linear) verifier with **aggregated proving**, but are left with one open issue. That is, the verification key is too large for on-chain deployment.

Searching for a commitment scheme to address this open issue does not land a definitive answer. Instead, it seems quite the opposite, that we might not be able to find such a scheme. Circuit constants are essentially matrixes of size O(N^2), however sparse. And as pointed out before, these matrixes may not be sparse any more after adopting aggregated proving.

Luckily there is a third option besides (1)committing to the circuit constants, or (2)turning to [Dory](https://eprint.iacr.org/2020/1274.pdf). That is, **recursive verification**. The idea is straightforward. Instead of verifying proofs from individual circuits directly, the OP_ZKP implementation has a verifier to verify proofs from one special circuit (the recursive circuit), which in turn recursively verifies proofs for business logic related circuits (application circuits).

The advantages are two fold:

(1) There is no need to deploy verification keys on-chain. The constants (and scheme-related public parameters) of the recursive verifier could be hard-coded in the Bitcoin client, taking up no block space. As for all application circuits, their circuit constants could be provided as witness to the recursive prover, thus hidden from the recursive verifier.

(2) It is possible to upgrade the recursive prover, the recursive verifier or the circuit without interrupting the applications based on OP_ZKP, in case of critical security patches. Still, it is each application circuit's responsibility to take care of their own security and efficiency.

Note that application circuits need not adopt the aggregated proving technique. As the verification is rather different with that of _normal_ circuits, it is unlikely to have the recursive verifier support both ways.

### Preliminary

We use bold lower case letters to denote vectors, for example, $\textbf{v} = (v\_0, v\_1, v\_2, ..., v\_{d-1})$, with explicit or implicit length $d$; bold number or its value representation to denote the vector of that number in increasing exponents, for example, $\textbf{2}$ = $(1, 2, 4, 8, ..., 2^{d-1})$ and $\textbf{x}\_\textbf{0}$ = $(1, x\_0, x\_0^2, ..., x\_0^{d-1})$; bold upper case letters to denote matrixes, for example, $\textbf{W}$. 

We use $[m]$ to denote close range $[1, m]$. $[\cdot]\cdot$ denotes scalar multiplication for an implicit EC group $\mathbb{G}$ of order $p$. $\langle \cdot, \cdot \rangle$ denotes the inner product operator, regardless of the types of the parameters: they could be either $\mathbb{Z}\_p \times \mathbb{Z}\_p \rightarrow \mathbb{Z}\_p$, or $\mathbb{Z}\_p \times \mathbb{G} \rightarrow \mathbb{G}$.

Throughout this document, the commitment scheme is Pedersen vector commitment. We also modify the Inner-Product Argument from [Bulletproofs](https://eprint.iacr.org/2017/1066.pdf) a bit following the [Halo paper](https://eprint.iacr.org/2019/1021.pdf), as one of the two vectors are usually known.

#### Pedersen Vector Commitment

For EC group $\mathbb{G}$ of order $p$, let $\textbf{G} \in \mathbb{G}^d, H \in \mathbb{G}$ be random generators. We commit to a vector $\textbf{v} \in \mathbb{Z}\_p^d$ as

$$\text{Commit}(\textbf{G}, H, \textbf{v}, r): C = \langle \textbf{v}, \textbf{G} \rangle + [r]H$$

where $r \in \mathbb{Z}\_p$ is a random blinder providing hiding.

We may ignore the public parameters $\textbf{G}$ and $H$ in the parameter list, and simply put

$$\text{Commit}(\textbf{v}, r): C = \langle \textbf{v}, \textbf{G} \rangle + [r]H \quad \quad \quad (1)$$

$\textbf{G}$ and $H$ should be fixed for Bitcoin use.

To open the commitment $(C, r)$, check if the identity holds

$$\text{Open}(C, \textbf{v}, r): C \overset{?}{=} \langle \textbf{v}, \textbf{G} \rangle + [r]H$$

#### Opening Commitments with Modified IPA

IPA reduces the communication cost of opening a Pedersen vector commitment from linear to log(n). We adopt a modified version of IPA based on [Halo](https://eprint.iacr.org/2019/1021.pdf), as in the case of opening a commitment $C$ of a polynomial $p(X)$, the opening point is usually provided by Verifier, thus known.

For polynomial $p(X)$ of degree less than $d$, let $\textbf{v}$ represents the coefficiencies of $p$: 

$$p(X) = \sum\_{i=0}^{d-1}v\_i \cdot X^i $$

Let its commitment value $C$ be defined according to Formula 1. Suppose $p$ evaluates to $a$ at $x\_0$:

$$a = p(x\_0) = \sum\_{i=0}^{d-1}v\_i \cdot x\_0^i = \langle \textbf{v}, \textbf{x}\_\textbf{0}\rangle$$

To open $C$ at $x\_0$ to value $a$, we use another random generator $U \in \mathbb{G}$, such that:

$$P = C + [a]U = \langle\textbf{v}, \textbf{G} \rangle + [r]H + [\langle \textbf{v}, \textbf{x}\_\textbf{0} \rangle]U \quad \quad \quad (2)$$

Instead of providing $\textbf{v}$ (and $r$) to open the above commitment, we reduce it to another commitment with half the size (assuming $d = 2^k$ for some $k > 0$). To do so, let $\textbf{v}\_{odd}$ and $\textbf{v}\_{even}$ denotes the odd-index part and even-index part of $\textbf{v}$, and similarly $\textbf{G}\_{odd}$ and $\textbf{G}\_{even}$, $\textbf{x}\_{0\_{odd}}$ and $\textbf{x}\_{0\_{even}}$:

$$\textbf{v}\_{odd} = (v\_1, v\_3, ..., v\_{d-1}) \quad \textbf{v}\_{even} = (v0, v2, ..., v\_{d-2})$$

$$\textbf{G}\_{odd} = (G\_1, G\_3, ..., G\_{d-1}) \quad \textbf{G}\_{even} = (G\_0, G\_2, ..., G\_{d-2})$$

$$\textbf{x}\_{0\_{odd}} = (x\_0, x\_0^3, ..., x\_0^{d-1}) \quad \textbf{x}\_{0\_{even}} = (1, x\_0^2, ..., x\_0^{d-2})$$

Given a challenge value $\gamma \in \mathbb{Z}\_p$, define new vectors of half the size:

$$\textbf{v}' = \textbf{v}\_{odd} + \gamma \cdot \textbf{v}\_{even}$$

$$\textbf{G}' = \textbf{G}\_{odd} + [\gamma^{-1}]\textbf{G}\_{even}$$

$$\textbf{x}\_\textbf{0}' = \textbf{x}\_{0\_{odd}} + \gamma^{-1} \cdot \textbf{x}\_{0\_{even}}$$

The new commitment value becomes

$$P' = \langle \textbf{v}', \textbf{G}' \rangle + [r]H + [\langle \textbf{v}', \textbf{x}\_\textbf{0}' \rangle]U$$

$$= P + [\gamma](\langle \textbf{v}\_{even}, \textbf{G}\_{odd} \rangle + [\langle \textbf{v}\_{even}, \textbf{x}\_{0\_{odd}} \rangle]U) + [\gamma^{-1}](\langle \textbf{v}\_{odd}, \textbf{G}\_{even} \rangle + [\langle \textbf{v}\_{odd}, \textbf{x}\_{0\_{even}} \rangle]U) $$

$$= P + [\gamma]L + [\gamma^{-1}]R$$

where $L$ and $R$ are provided from Prover to Verifier before the latter generating the challenge value $\gamma$:

$$L = \langle \textbf{v}\_{even}, \textbf{G}\_{odd} \rangle + [\langle \textbf{v}\_{even}, \textbf{x}\_{0\_{odd}} \rangle]U$$

$$R = \langle \textbf{v}\_{odd}, \textbf{G}\_{even} \rangle + [\langle \textbf{v}\_{odd}, \textbf{x}\_{0\_{even}} \rangle]U$$

Based on the values $P, L, R, \gamma$, Verifier could calculate $P'$ on its own.

Rename $(P', \textbf{G}', \textbf{v}', \textbf{x}'\_\textbf{0})$ to $(P, \textbf{G}, \textbf{v}, \textbf{x}\_\textbf{0})$. Repeat this process till $\textbf{v}'$ contains only one element, than Prover can directly provide its value to Verifier. Verifier checks if the commitment opens correctly.

*Remark* Chosing odd/even rather than lower/higher halves is based on the need to align many sub-circuits with different size.

#### Batched Opening

##### Same Polynomial, Multiple Points
First we describe how to open a commitment value $C$ to a polynomial $p(X)$ at multiple points $(x\_1, x\_2, ..., x\_m)$ with IPA. Suppose we have $a\_i = p(x\_i), i \in [m]$, we need to open $C$ at each $x\_i$ to $a\_i$. 

Let $U\_i, i \in [m]$ be $m$ random generators. At the begining we shall have 

$$P = C + \sum\_{i=1}^m[{a\_i}]U\_i = \langle\textbf{v}, \textbf{G} \rangle + [r]H + \sum\_{i=1}^m[\langle \textbf{v}, \textbf{x}\_\textbf{i} \rangle]U\_i \quad \quad \quad (3)$$

For each round, in a similar way we may have:
$$\textbf{x}\_\textbf{i}' = \textbf{x}\_{i\_{odd}} + \gamma^{-1} \cdot \textbf{x}\_{i\_{even}}$$

and

$$P' = P + [\gamma]L + [\gamma^{-1}]R$$

$$L = \langle \textbf{v}\_{even}, \textbf{G}\_{odd} \rangle + \sum\_{i=1}^m[\langle \textbf{v}\_{even}, \textbf{x}\_{i\_{odd}} \rangle]U\_i$$

$$R = \langle \textbf{v}\_{odd}, \textbf{G}\_{even} \rangle + \sum\_{i=1}^m[\langle \textbf{v}\_{odd}, \textbf{x}\_{i\_{even}} \rangle]U\_i$$

Besides necessary values of $a\_i$, there are no extra communication costs.

##### Different Polynomial, Same Point

Next we describe how to open multiple polynomials at the same point. Suppose we have $m$ polynomials $p\_i(X)$ for $i \in [m]$, each is evaluated to $a\_i$ at the same point $x\_0$:

$$a\_i = p\_i(x\_0), i \in [m]$$

Prover first need to commit to each of the polynomials. Let the commitments be $C\_i, i \in [m]$. For simplicity, let's use $\textbf{p}\_i$ instead of $\textbf{v}$ to represent the the vector of coefficiencies of polynomial $p\_i(X)$. We have:

$$C\_i = \text{Commit}(\textbf{p}\_i, r\_i) = \langle \textbf{p}\_i, \textbf{G} \rangle + [r\_i]H$$

Verifier generates a random value as challenge: $\beta \in \mathbb{Z}\_p$. Prover computes:

$$C = \sum\_{i=1}^m \beta^i \cdot C\_i = \sum\_{i=1}^m \langle \beta^i \cdot \textbf{p}\_i, \textbf{G} \rangle + [\beta^i \cdot r\_i]H \quad \quad \quad (4)$$

Equivalently there exists a batched polynomial

$$p(X) = \sum\_{i=1}^m \beta^i \cdot p\_i(X)$$

which is commited to $C$, with $r = \sum\_{i=1}^m \beta^i \cdot r\_i$ as blinder:

$$C = \text{Commit}(\textbf{p}, r)$$

And this commitment value $C$ shall be open to $a = \sum\_{i=1}^m \beta^i \cdot a\_i$ at $x\_0$. For that we already have a protocol.

##### Hybrid Case

We might have hybrid case for the Tea/Horse proving system: opening $R$ for $r(X, 1)$ at two different points $z$ and $yz$, and opening $T\_{lo}$ for $t\_{lo}(X, y)$ and $T\_{hi}$ for $t\_{hi}(X, y)$ at $z$. A combination of the previous two cases suffices here. We leave the details to later sections.

Note that the communication costs here are three $\mathbb{G}$ elements for commitments and four $\mathbb{Z}\_p$ elements for open-to values, in addition to what is required by IPA. The verifier complexity is still linear to the polynomial size.

### The Proving System

In the Tea/Horse proving system, a typical R1CS circuit is organized as many (m) sub-circuits, each having a size of fixed upbound set to $2^{16}$. In this section, we describe how the system works.

The basic idea is to aggregatedly prove all the sub-circuits. For each sub-circuit, we follow a protocol which is a mixture of Sonic and Halo. Then we have an argument to aggregate them together.

#### Sonic Arithemtic

In this section we reiterate the Sonic Arithmetic following notations from [Sonic Paper](https://eprint.iacr.org/2019/099.pdf) and [Halo paper](https://eprint.iacr.org/2019/1021.pdf) with slight modifications introducing matrix notations. We assume that the circuit has been preprocessed (see Appendix A of [Bootleproofs](https://eprint.iacr.org/2016/263.pdf)).

Suppose we have an arithmetic circuit $C$ and let $N$, $Q$, $d$, $k$ be integers such that $d = 4N = 2^k$ and $3Q \lt d$. There are $N$ multiplication gates such that the $\textit{i}$-th constaint is:

$$a\_i \cdot b\_i = c\_i \quad \quad \quad (5)$$

where $a\_i, b\_i, c\_i$ are the $i$-th element of the witnesses vectors $\textbf{a}, \textbf{b}, \textbf{c} \in \mathbb{Z}\_p^{N}$.

For the $Q$ linear constraints capturing the copy wires, multiplied-by-constant gates and addition gates, we denote $\textbf{U}, \textbf{V}, \textbf{W} \in \mathbb{Z}\_p^{Q \times N}$ and $\textbf{k} \in \mathbb{Z}\_p^Q$ such that:

$$\textbf{U}\cdot\textbf{a} + \textbf{V}\cdot\textbf{b} + \textbf{W}\cdot\textbf{c} = \textbf{k} \quad \quad \quad (6)$$

Here, $\textbf{U}, \textbf{V}, \textbf{W}, \textbf{k}$ are the circuit constants (where there is no confusion, we use row vector and column vector interchangedly).

For $\textbf{M} \in \\{\textbf{U}, \textbf{V}, \textbf{W}\\}$, we denote 

$$m\_i(Y) = \sum\_{q=1}^Q Y^q\cdot M\_{q,i} $$

where $M\_{q,i}$ denotes the element at the position of $(q, i)$ of matrix $\textbf{M}$. 
And let 

$$k(Y) = \sum_{q=1}^Q Y^q \cdot k_q$$

Embedding all the constraints into a single equation of $Y$, we have

$$ Y^N (\sum_{i=1}^N (a_i u_i(Y) + b_i v_i(Y) + c_i w_i(Y)) - k(Y)) + \sum_{i=1}^N (a_i b_i - c_i)\cdot (Y^i + Y^{-i}) = 0 \quad \quad \quad (7)$$

which should hold at all points if all the constraint system is satisfied for some witness $\textbf{a, b, c} \in \mathbb{Z}\_p^N$. Given a large enough field, this means the constraint system is satisfied with high probability if Equation 7 holds for a random challenge value $y \in \mathbb{Z}\_p$ and some witness values.

Going on with Sonic Arithmetic, let's define some more polynomials:

$$r(X, Y) = \sum\_{i=1}^N(a\_iX^iY^i + b\_iX^{-i}Y^{-i} + c\_iX^{-i-N}Y^{-i-N})$$

$$s(X, Y) = \sum\_{i=1}^N(u\_i(Y)X^{-i} + v\_i(Y)X^i + w\_i(Y)X^{i+N})$$

$$s'(X, Y) = Y^Ns(X, Y) - \sum\_{i=1}^N(Y^i + Y^{-i})X^{i+N}$$

$$t(X, Y) = r(X, 1)(r(X, Y) + s'(X, Y)) - Y^Nk(Y) \quad \quad \quad (8)$$

Note that the constant term of $t(X, Y)$ in terms of $X$, that is, the coefficient of $X^0$, is exactly the left-hand side of Equation 7. So the protocol design is to show that, Equation 8 still holds after removing the constant term from the left-hand side.

#### Single Sub-Circuit Proving

This part is still mostly Sonic Arithmetization with (modified) IPA. Some ideas come from [Halo](https://eprint.iacr.org/2019/1021.pdf) but we do not use the amortizing part or cycles of curves part. We stick to the `secp256k1` curve. Also as we hard-coded some circuit constants in the verifier side, verifier could compute these values on its own. 

Prover commits to $r(X, 1)$ with blinder $\delta\_R$ for circuit $C$. We follow [Halo paper](https://eprint.iacr.org/2019/1021.pdf) to scale it with $X^{3N-1}$ to ensure $r(X, 1)$ is at most degreen $N$.

$$\delta\_R \overset{\\$}{\leftarrow} \mathbb{Z}\_p,\quad R \leftarrow \text{Commit}(r(X, 1)X^{3N-1}; \delta\_R) $$

Verifier responds with a challenge value $y \in \mathbb{Z}\_p$. In a non-interactive setting, $y$ should be generated by hashing the past transcriptions up to and including the commitment value $R$. We ignore this notion in later sections.

Now let $t\_{lo}(X, y), t\_0(y), t\_{hi}(X, y)$ be the part of $t(X, y)$ with negative exponent, constant term, and with positive exponent, for X, that is,

$$t(X, y) = t\_{lo}(X, y)X^{-d} + t\_0(y) + t\_{hi}(X, y)X$$

Prover commits to $t\_{lo}$ and $t\_{hi}$.

$$\delta\_{T\_{lo}} \overset{\\$}{\leftarrow} \mathbb{Z}\_p,\quad T\_{lo} \leftarrow \text{Commit}(t\_{lo}(X, y); \delta\_{T\_{lo}}) $$

$$\delta\_{T\_{hi}} \overset{\\$}{\leftarrow} \mathbb{Z}\_p,\quad T\_{hi} \leftarrow \text{Commit}(t\_{hi}(X, y); \delta\_{T\_{hi}}) $$

Verifier responds with a challenge value $z \in \mathbb{Z}\_p$.

Prover now needs to open a few commitments, and provides open to values and proofs.

$$(e = r(z, 1), proof\_a) \leftarrow \text{Open}\_\text{IPA}(R, z, r(X, 1), \delta\_R)$$

$$(f = r(z, y), proof\_b) \leftarrow \text{Open}\_\text{IPA}(R, yz, r(X, 1), \delta\_R)$$

$$(t\_1 = t\_{lo}(z, y), proof\_{t\_{lo}}) \leftarrow \text{Open}\_\text{IPA}(T\_{lo}, z, t\_{lo}(X, y), \delta\_{T\_{lo}})$$

$$(t\_2 = t\_{hi}(z, y), proof\_{t\_{hi}}) \leftarrow \text{Open}\_\text{IPA}(T\_{hi}, z, t\_{hi}(X, y), \delta\_{T\_{hi}})$$

Verifier need to verify all the above openings, compute $s = s(z, y)$, $s'$ from $s$, $k = k(y)$, then check if
$$t\_1 + t\_2 \overset{?}{=} e(f + s') - k \quad \quad \quad (9)$$

Passing Identity Test 9 means that $t_0(y) = 0$ with overwhelming probability. If all checks pass, Verifier returns 1, otherwise 0.

*Remark* The computation of $s$ does not involve confidential witness data $\textbf{a}, \textbf{b}$ or $\textbf{c}$, so Verifier could compute on its own. Opening a commitment to $s(X, y)$ is not viable for aggregated proving, as direct computation in $\mathbb{Z}\_p$ should be much faster than verifying opening with IPA. Also the size of none-zero entries in the aggregated $\textbf{U}, \textbf{V}$ or $\textbf{W}$ might exceed $d$. That's the reason we don't bother committing to it even for single sub-circuit. We will discuss the optimization for computing $s$ in later sections.

##### Batched Opening for Sub-Circuit

The opening of $R$ at $z$, $yz$, of $T\_{lo}$ and $T\_{hi}$ at $z$, could be batched together with the batching techniques we have described in the last section.

Specfically, $R$ could be opened to $e \in \mathbb{Z}\_p$ and $f \in \mathbb{Z}\_p$ at $z$ and $yz$ given random generators $U\_1$ and $U\_2$. We have

$$P\_R = R + [e]U\_1 + [f]U\_2 = \langle\textbf{r}, \textbf{G} \rangle + [\delta\_R]H + [\langle \textbf{r}, \textbf{z} \rangle]U\_1 + [\langle \textbf{r}, \textbf{zy} \rangle]U\_2$$

where $\textbf{r}$ denotes the coefficiencies for $r(X, 1)$.

To open $T\_{lo}$ and $T\_{hi}$ at $z$ in a batch along with $R$, given $\beta$ as the challenge value, we should have

$$R + [\beta]T\_{lo} + [\beta^2]T\_{hi} + [e + \beta \cdot t\_1 + \beta^2 \cdot t\_2]U\_1 + [f]U\_2$$

$$\quad = \langle\textbf{r} + \beta \cdot \textbf{t}\_\textbf{lo} + \beta^2 \cdot \textbf{t}\_\textbf{hi}, \textbf{G} \rangle + [\delta\_R + \beta \cdot \delta\_{lo} + \beta^2 \cdot \delta\_{hi}]H $$

$$\quad \quad + [\langle \textbf{r} + \beta \cdot \textbf{t}\_\textbf{lo} + \beta^2 \cdot \textbf{t}\_\textbf{hi}, \textbf{z} \rangle]U\_1 + [\langle \textbf{r}, \textbf{zy} \rangle]U\_2 \quad \quad \quad (10)$$

#### Aggregated Proving

We now consider how we could aggregate the proving of $m$ circuits together, denoted $C^{(i)}$ for $i \in [m]$. We use superscript $(i)$ to indicate the $i$-th circuit. We have $N$ multiplication constraints for sub-circuit $C^{(i)}$

$$\textbf{a}^{(i)} \circ \textbf{b}^{(i)} = \textbf{c}^{(i)}$$

where $\circ$ denotes the element-wise multiplication, or Hadamard product, for witnesses $\textbf{a}^{(i)}, \textbf{b}^{(i)}, \textbf{c}^{(i)} \in \mathbb{Z}\_p^N$.

And $Q$ linear constraints:

$$\textbf{U}^{(i)} \cdot \textbf{a}^{(i)} + \textbf{V}^{(i)} \cdot \textbf{b}^{(i)} + \textbf{W}^{(i)} \cdot \textbf{c}^{(i)} = \textbf{k}^{(i)}$$

where $\textbf{U}^{(i)}, \textbf{V}^{(i)}, \textbf{W}^{(i)} \in \mathbb{Z}\_p^{Q \times N}, \textbf{k}^{(i)} \in \mathbb{Z}\_p^Q$ are constants for $C^{(i)}$.

Finally we can define polynomials $r^{(i)}(X, Y)$, $s^{(i)}(X, Y)$ and $t^{(i)}(X, Y)$ for circuit $C^{(i)}$. Before aggregation could happen, Prover needs to commit to each $r^{(i)}(X, 1)$: 

$$\delta_R^{(i)} \overset{\\$}{\leftarrow} \mathbb{Z}_p,\quad R^{(i)} \leftarrow \text{Commit}(r^{(i)}(X, 1)X^{3N-1}; \delta_R^{(i)}) $$

and Verifier responds with a challenge value $y \in \mathbb{Z}\_p$. Note that all sub-circuit shares the same $y$ value.

Next, Prover may commit to each of $t\_{lo}^{(i)}(X, y)$ and $t\_{hi}^{(i)}(X, y)$:

$$\delta\_{T\_{lo}^{(i)}} \overset{\\$}{\leftarrow} \mathbb{Z}\_p,\quad T\_{lo}^{(i)} \leftarrow \text{Commit}(t\_{lo}^{(i)}(X, y); \delta\_{T\_{lo}^{(i)}}) $$

$$\delta\_{T\_{hi}^{(i)}} \overset{\\$}{\leftarrow} \mathbb{Z}\_p,\quad T\_{hi}^{(i)} \leftarrow \text{Commit}(t\_{hi}^{(i)}(X, y); \delta\_{T\_{hi}^{(i)}}) $$

Verifier responds with a challenge value $z \in \mathbb{Z}\_p$, which holds the same for all sub-circuits. And further two more challenge value $\alpha, \beta \in \mathbb{Z}\_p$. We use $\alpha$ to combine polynomials, commitment values, blinders and open-to values together:

$$r(X, 1) = \sum_{i=1}^m \alpha^i \cdot r^{(i)}(X, 1) \quad \quad s(X, y)= \sum_{i=1}^m \alpha^i \cdot s^{(i)}(X, y) $$

$$t\_{lo}(X, y)= \sum\_{i=1}^m \alpha^i \cdot t\_{lo}^{(i)}(X, y) \quad \quad t\_{hi}(X, y)= \sum\_{i=1}^m \alpha^i \cdot t\_{hi}^{(i)}(X, y) $$

$$R = \sum\_{i=1}^m \alpha^i \cdot R^{(i)} \quad \quad T\_{lo} = \sum\_{i=1}^m \alpha^i \cdot T\_{lo}^{(i)} \quad \quad T\_{hi} = \sum\_{i=1}^m \alpha^i \cdot T\_{hi}^{(i)}$$

$$\delta\_R = \sum\_{i=1}^m \alpha^i \cdot \delta\_{R^{(i)}} \quad \quad \delta\_{T\_{lo}} = \sum\_{i=1}^m \alpha^i \cdot \delta\_{T\_{lo}^{(i)}} \quad \quad \delta\_{T\_{hi}} = \sum\_{i=1}^m \alpha^i \cdot \delta\_{T\_{hi}^{(i)}}$$

$$e = \sum\_{i=1}^m\alpha^i\cdot e^{(i)}\quad f = \sum\_{i=1}^m\alpha^i\cdot f^{(i)}\quad t\_1 = \sum\_{i=1}^m\alpha^i\cdot t\_1^{(i)}\quad t\_2 = \sum\_{i=1}^m\alpha^i\cdot t\_2^{(i)}$$

Then we use $\beta$ to batch open the combined polynomials, applying the Equation 10 with the definitions given above.

After checking that Equation 10 holds, Verifier compute $s'$ and $k$, and check if the Identity Test 9 passes.

#### Costs Analysis

For $m$ sub-ircuits each with $N$ multiplication constraints, the proof data include:

- commitments to $r^{(i)}(X, 1), t_{lo}^{(i)}(X, y), t_{hi}^{(i)}(X, y)$, totally $3m$ elements of $\mathbb{G}$, denoted $3m[G]$;
- aggregated opening hints $\delta_R, \delta_{T_{lo}}, \delta_{T_{hi}}$, totally 3 elements of $\mathbb{Z}_p$, denoted $3[F]$;
- aggregated open-to values $e, f, t_1, t_2$, totally 4 elements of $\mathbb{Z}_p$, denoted $4[F]$;
- proof data for opening the aggregated commitment, which is $2log_2(d)[G] + 1[F] = (2log_2(N) + 4)[G] + 1[F] $ (remember $d = 4N$). 

And the verification key include:

- elements of combined $\textbf{U}, \textbf{V}, \textbf{W}$, of apparent size $3QN$, which could be further optimized;
- elements of combined $\textbf{k}$, of size $Q$.

Verifier complexity is:

- combining $3m$ commitments to 3 commitments, that is $3$ multi-scalar multiplications, echo of size $m$;
- batch opening the combined commitment, dominated by $d = 4N$ sized multi-scalar multiplication;
- computing the aggregated $s(X, Y)$ and $k(Y)$, of apparent $O(N^2)$ field operations in $\mathbb{Z}_p$, which could be further optimized.

#### Optimization

#### Data Saving

First observe that combined $\textbf{U}, \textbf{V}, \textbf{W}$ are roughly triangle matrixes, with the upper right half mostly zero. This is due to the fact that linear constraints can only reference existing wires. This cuts half the storage cost and runtime for the verifier.

Next observation is that some circuits are dominated by a few sub-circuits repeated many times. For example, based on the cost analysis above, the recursive verifier for a Tea/Horse proof is dominated by multi-scalar multiplicatioins and some field operations. In that case, the same set of $\{\textbf{U}_0, \textbf{V}_0, \textbf{W}_0\}$ could be used for many sets of data of the same sub-circuit. So the aggregated $\textbf{U}, \textbf{V}, \textbf{W}$ could still be sparse. We conjecture that the actual cost could be more likely $O(N)$ rather than $O(N^2)$, although the constant factor might be concretely large, say, a few tens or hundred.

##### Further Optimizaiton - First Attempt

Last we dicsuss the feasibility of proving the existence of the $3m$ commitments for $r^{(i)}(X, 1), t_{lo}^{(i)}(X, y), t_{hi}^{(i)}(X, y)$, with a dedicated circuit. This is the idea borrowed from [Nova](https://eprint.iacr.org/2021/370.pdf), which employs zkSnark to prove the folding process.

_Remark_ Note that Nova in itself cannot be applied here as it requires cycles of elliptic curves. Also Nova requires auxillary zkSnark circuits, which has similar performance characteristics as analyzed below.

For commitment values $R^{(i)}, T_{lo}^{(i)}, T_{hi}^{(i)}, i \in [m]$, we need a circuit to prove the following computation:

- derive a value $\alpha_C \in \mathbb{Z}_p^*$ by hashing $C^{(i)}$ together for $C \in \{R, T_{lo}, T_{hi}\}$. A field friendly hash would be helpful. However we might need just `Sha256` for Bitcoin.
- compute $\alpha = \alpha_R \cdot \alpha_{T_{lo}} \cdot \alpha_{T_{hi}}$
- use the derived $\alpha$ to combined the commitment values $C = \sum_{i=1}^m\alpha^iC^{(i)}$ for $C \in \{R, T_{lo}, T_{hi}\}$

Verifier verifies the proof of the above computation, instead of computing $R, T_{lo}, T_{hi}$ on his own. The $3m$ commitment values $R^{(i)}, T_{lo}^{(i)}, T_{hi}^{(i)}$ are replaced with the proof data of the above circuit. So wow large would this circuit be, and the proof data?

The circuit needs to 

- hash $3m$ elements of $\mathbb{G}$
- run $3$ multi-scalar multiplication, each of size $m$

Unfortunately, it costs much more to run MSM in-circiut than directly, both proving and verification with IPA.

##### Second Attempt

Take a step back. We could start from $r^{(i)}(X, 1), t_{lo}^{(i)}(X, y), t_{hi}^{(i)}(X, y)$ directly and prove correct derivation of $\alpha$ and aggregation of $r(X, 1), t_{lo}(X, y), t_{hi}(X, y)$, etc. For $m$ sub-circuits each of size $O(d)$, totally there are $O(dm)$ elements of $\mathbb{Z}_p$ to process. The costs with IPA are $O(log_2(d) + log_2(m))$ for proof data, $O(dm)$ size MSM for verification runtime, which is much more expensive than direct computation.

##### Review and Alternatives

If we set $N$ to $2^{16}$, then $m$ might range from a few donzens to a few thousands for reasonable circuits. Under this assumption, Tea/Horse proving system has shorter proof yet longer runtime than [Hyrax](https://eprint.iacr.org/2017/1132.pdf), which typically features $O(\sqrt{n})$ proof size and verifier complexity.

At this point our major issues are (1) with the verification key size, which is $O(N)$ with a big constant factor under our conjecture; and (2), the size of the recursive verifier, which could be very large, given it has to compute a large MSM and many field operations proportional to verification key size.

The application circuits have to be in a scheme with succinct verifier, for example Tea/Horse or Hyrax. But even with Hyrax the recursive verifier is still very large.

#### Sparse Polynomial Commitment

Now let's take a step back and review if we can apply sparse polynomial commitment technique, called Spark, which was introduced in [Spartan](https://eprint.iacr.org/2019/550.pdf) and refined in [Lasso](https://people.cs.georgetown.edu/jthaler/Lasso-paper.pdf).

Specifically, we seek to remove the circuit constants from the verifier key, replace them with some commitments. Then we don't need a recursive verifier, the proposed OP_ZKP implementation could verify proof data of application circuits directly. This addresses the two issues mentioned above.

_Remark_ This is a live document. We might change our mind or switch to a better solution in the middle of developing the system.