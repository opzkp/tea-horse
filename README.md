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

We use bold lower case letters to denote vectors, for example, $\textbf{v} = (v_0, v_1, v_2, ..., v_{d-1})$, with explicit or implicit length $d$; bold number or its value representation to denote the vector of that number in increasing exponents, for example, $\textbf{2}$ = $(1, 2, 4, 8, ..., 2^{d-1})$ and $\textbf{x}_\textbf{0}$ = $(1, x_0, x_0^2, ..., x_0^{d-1})$; bold upper case letters to denote matrixes, for example, $\textbf{W}$. 

We use $[m]$ to denote close range $[1, m]$. $[\cdot]\cdot$ denotes scalar multiplication for an implicit EC group $\mathbb{G}$ of order $p$. $\langle \cdot, \cdot \rangle$ denotes the inner product operator, regardless of the types of the parameters: they could be either $\mathbb{Z}_p \times \mathbb{Z}_p \rightarrow \mathbb{Z}_p$, or $\mathbb{Z}_p \times \mathbb{G} \rightarrow \mathbb{G}$.

Throughout this document, the commitment scheme is Pedersen vector commitment. We also modify the Inner-Product Argument from [Bulletproofs](https://eprint.iacr.org/2017/1066.pdf) a bit following the [Halo paper](https://eprint.iacr.org/2019/1021.pdf), as one of the two vectors are usually known.

#### Pedersen Vector Commitment

For EC group $\mathbb{G}$ of order $p$, let $\textbf{G} \in \mathbb{G}^d, H \in \mathbb{G}$ be random generators. We commit to a vector $\textbf{v} \in \mathbb{Z}_p^d$ as
$$\text{Commit}(\textbf{G}, H, \textbf{v}, r): C = \langle \textbf{v}, \textbf{G} \rangle + [r]H$$
where $r \in \mathbb{Z}_p$ is a random blinder providing hiding.

We may ignore the public parameters $\textbf{G}$ and $H$ in the parameter list, and simply put
$$\text{Commit}(\textbf{v}, r): C = \langle \textbf{v}, \textbf{G} \rangle + [r]H \tag{1}$$
$\textbf{G}$ and $H$ should be fixed for Bitcoin use.

To open the commitment $(C, r)$, check if the identity holds
$$\text{Open}(C, \textbf{v}, r): C \overset{?}{=} \langle \textbf{v}, \textbf{G} \rangle + [r]H$$

#### Opening Commitments with Modified IPA

IPA reduces the communication cost of opening a Pedersen vector commitment from linear to log(n). We adopt a modified version of IPA based on [Halo](https://eprint.iacr.org/2019/1021.pdf), as in the case of opening a commitment $C$ of a polynomial $p(X)$, the opening point is usually provided by Verifier, thus known.

For polynomial $p(X)$ of degree less than $d$, let $\textbf{v}$ represents the coefficiencies of $p$: 
$$p(X) = \sum_{i=0}^{d-1}v_i \cdot X^i $$

Let its commitment value $C$ be defined according to Formula 1. Suppose $p$ evaluates to $a$ at $x_0$:

$$a = p(x_0) = \sum_{i=0}^{d-1}v_i \cdot x_0^i = \langle \textbf{v}, \textbf{x}_\textbf{0}\rangle$$

To open $C$ at $x_0$ to value $a$, we use another random generator $U \in \mathbb{G}$, such that:
$$P = C + [a]U = \langle\textbf{v}, \textbf{G} \rangle + [r]H + [\langle \textbf{v}, \textbf{x}_\textbf{0} \rangle]U \tag{2}$$
Instead of providing $\textbf{v}$ (and $r$) to open the above commitment, we reduce it to another commitment with half the size (assuming $d = 2^k$ for some $k > 0$). To do so, let $\textbf{v}_{odd}$ and $\textbf{v}_{even}$ denotes the odd-index part and even-index part of $\textbf{v}$, and similarly $\textbf{G}_{odd}$ and $\textbf{G}_{even}$, $\textbf{x}_{0_{odd}}$ and $\textbf{x}_{0_{even}}$:
$$\textbf{v}_{odd} = (v_1, v_3, ..., v_{d-1}) \quad \textbf{v}_{even} = (v0, v2, ..., v_{d-2})$$
$$\textbf{G}_{odd} = (G_1, G_3, ..., G_{d-1}) \quad \textbf{G}_{even} = (G_0, G_2, ..., G_{d-2})$$
$$\textbf{x}_{0_{odd}} = (x_0, x_0^3, ..., x_0^{d-1}) \quad \textbf{x}_{0_{even}} = (1, x_0^2, ..., x_0^{d-2})$$

Given a challenge value $\gamma \in \mathbb{Z}_p$, define new vectors of half the size:
$$\textbf{v}' = \textbf{v}_{odd} + \gamma \cdot \textbf{v}_{even}$$
$$\textbf{G}' = \textbf{G}_{odd} + [\gamma^{-1}]\textbf{G}_{even}$$
$$\textbf{x}_\textbf{0}' = \textbf{x}_{0_{odd}} + \gamma^{-1} \cdot \textbf{x}_{0_{even}}$$

The new commitment value becomes
$$P' = \langle \textbf{v}', \textbf{G}' \rangle + [r]H + [\langle \textbf{v}', \textbf{x}_\textbf{0}' \rangle]U$$
$$= P + [\gamma](\langle \textbf{v}_{even}, \textbf{G}_{odd} \rangle + [\langle \textbf{v}_{even}, \textbf{x}_{0_{odd}} \rangle]U) + [\gamma^{-1}](\langle \textbf{v}_{odd}, \textbf{G}_{even} \rangle + [\langle \textbf{v}_{odd}, \textbf{x}_{0_{even}} \rangle]U) $$
$$= P + [\gamma]L + [\gamma^{-1}]R$$
where $L$ and $R$ are provided from Prover to Verifier before the latter generating the challenge value $\gamma$:
$$L = \langle \textbf{v}_{even}, \textbf{G}_{odd} \rangle + [\langle \textbf{v}_{even}, \textbf{x}_{0_{odd}} \rangle]U$$
$$R = \langle \textbf{v}_{odd}, \textbf{G}_{even} \rangle + [\langle \textbf{v}_{odd}, \textbf{x}_{0_{even}} \rangle]U$$

Based on the values $P, L, R, \gamma$, Verifier could calculate $P'$ on its own.

Rename $(P', \textbf{G}', \textbf{v}', \textbf{x}'_\textbf{0})$ to $(P, \textbf{G}, \textbf{v}, \textbf{x}_\textbf{0})$. Repeat this process till $\textbf{v}'$ contains only one element, than Prover can directly provide its value to Verifier. Verifier checks if the commitment opens correctly.

*Remark* Chosing odd/even rather than lower/higher halves is based on the need to align many sub-circuits with different size.

#### Batched Opening

##### Same Polynomial, Multiple Points
First we describe how to open a commitment value $C$ to a polynomial $p(X)$ at multiple points $(x_1, x_2, ..., x_m)$ with IPA. Suppose we have $a_i = p(x_i), i \in [m]$, we need to open $C$ at each $x_i$ to $a_i$. 

Let $U_i, i \in [m]$ be $m$ random generators. At the begining we shall have 
$$P = C + \sum_{i=1}^m[{a_i}]U_i = \langle\textbf{v}, \textbf{G} \rangle + [r]H + \sum_{i=1}^m[\langle \textbf{v}, \textbf{x}_\textbf{i} \rangle]U_i \tag{3}$$

For each round, in a similar way we may have:
$$\textbf{x}_\textbf{i}' = \textbf{x}_{i_{odd}} + \gamma^{-1} \cdot \textbf{x}_{i_{even}}$$
and
$$P' = P + [\gamma]L + [\gamma^{-1}]R$$
$$L = \langle \textbf{v}_{even}, \textbf{G}_{odd} \rangle + \sum_{i=1}^m[\langle \textbf{v}_{even}, \textbf{x}_{i_{odd}} \rangle]U_i$$
$$R = \langle \textbf{v}_{odd}, \textbf{G}_{even} \rangle + \sum_{i=1}^m[\langle \textbf{v}_{odd}, \textbf{x}_{i_{even}} \rangle]U_i$$

Besides necessary values of $a_i$, there are no extra communication costs.

##### Different Polynomial, Same Point

Next we describe how to open multiple polynomials at the same point. Suppose we have $m$ polynomials $p_i(X)$ for $i \in [m]$, each is evaluated to $a_i$ at the same point $x_0$:
$$a_i = p_i(x_0), i \in [m]$$

Prover first need to commit to each of the polynomials. Let the commitments be $C_i, i \in [m]$. For simplicity, let's use $\textbf{p}_i$ instead of $\textbf{v}$ to represent the the vector of coefficiencies of polynomial $p_i(X)$. We have:
$$C_i = \text{Commit}(\textbf{p}_i, r_i) = \langle \textbf{p}_i, \textbf{G} \rangle + [r_i]H$$

Verifier generates a random value as challenge: $\beta \in \mathbb{Z}_p$. Prover computes:
$$C = \sum_{i=1}^m \beta^i \cdot C_i = \sum_{i=1}^m \langle \beta^i \cdot \textbf{p}_i, \textbf{G} \rangle + [\beta^i \cdot r_i]H \tag{4}$$
Equivalently there exists a batched polynomial
$$p(X) = \sum_{i=1}^m \beta^i \cdot p_i(X)$$
which is commited to $C$, with $r = \sum_{i=1}^m \beta^i \cdot r_i$ as blinder:
$$C = \text{Commit}(\textbf{p}, r)$$

And this commitment value $C$ shall be open to $a = \sum_{i=1}^m \beta^i \cdot a_i$ at $x_0$. For that we already have a protocol.

##### Hybrid Case

We might have hybrid case for the Tea/Horse proving system: opening $R$ for $r(X, 1)$ at two different points $z$ and $yz$, and opening $T_{lo}$ for $t_{lo}(X, y)$ and $T_{hi}$ for $t_{hi}(X, y)$ at $z$. A combination of the previous two cases suffices here. We leave the details to later sections.

Note that the communication costs here are three $\mathbb{G}$ elements for commitments and four $\mathbb{Z}_p$ elements for open-to values, in addition to what is required by IPA. The verifier complexity is still linear to the polynomial size.

### The Proving System

In the Tea/Horse proving system, a typical R1CS circuit is organized as many (m) sub-circuits, each having a size of fixed upbound set to $2^{16}$. In this section, we describe how the system works.

The basic idea is to aggregatedly prove all the sub-circuits. For each sub-circuit, we follow a protocol which is a mixture of Sonic and Halo. Then we have an argument to aggregate them together.

#### Sonic Arithemtic

In this section we reiterate the Sonic Arithmetic following notations from [Sonic Paper](https://eprint.iacr.org/2019/099.pdf) and [Halo paper](https://eprint.iacr.org/2019/1021.pdf) with slight modifications introducing matrix notations. We assume that the circuit has been preprocessed (see Appendix A of [Bootleproofs](https://eprint.iacr.org/2016/263.pdf)).

Suppose we have an arithmetic circuit $C$ and let $N$, $Q$, $d$, $k$ be integers such that $d = 4N = 2^k$ and $3Q \lt d$. There are $N$ multiplication gates such that the $\textit{i}$-th constaint is:

$$ a_i \cdot b_i = c_i \tag{5}$$

where $a_i, b_i, c_i$ are the $i$-th element of the witnesses vectors $\textbf{a}, \textbf{b}, \textbf{c} \in \mathbb{Z}_p^{N}$.

For the $Q$ linear constraints capturing the copy wires, multiplied-by-constant gates and addition gates, we denote $\textbf{U}, \textbf{V}, \textbf{W} \in \mathbb{Z}_p^{Q \times N}$ and $\textbf{k} \in \mathbb{Z}_p^Q$ such that:

$$\textbf{U}\cdot\textbf{a} + \textbf{V}\cdot\textbf{b} + \textbf{W}\cdot\textbf{c} = \textbf{k} \tag{6}$$

Here, $\textbf{U}, \textbf{V}, \textbf{W}, \textbf{k}$ are the circuit constants (where there is no confusion, we use row vector and column vector interchangedly).

For $\textbf{M} \in \{\textbf{U}, \textbf{V}, \textbf{W}\}$, we denote 
$$m_i(Y) = \sum_{q=1}^Q Y^q\cdot M_{q,i} $$
where $M_{q,i}$ denotes the element at the position of $(q, i)$ of matrix $\textbf{M}$. And let $$ k(Y) = \sum_{q=1}^Q Y^q \cdot k_q$$
Embedding all the constraints into a single equation of $Y$, we have

$$ Y^N (\sum_{i=1}^N(a_iu_i(Y) + b_iv_i(Y) + c_iw_i(Y)) - k(Y)) + \sum_{i=1}^N(a_i b_i - c_i)\cdot(Y^i + Y^{-i}) = 0 \tag{7}$$

which should hold at all points if all the constraint system is satisfied for some witness $\textbf{a, b, c} \in \mathbb{Z}_p^N$. Given a large enough field, this means the constraint system is satisfied with high probability if Equation 7 holds for a random challenge value $y \in \mathbb{Z}_p$ and some witness values.

Going on with Sonic Arithmetic, let's define some more polynomials:

$$r(X, Y) = \sum_{i=1}^N(a_iX^iY^i + b_iX^{-i}Y^{-i} + c_iX^{-i-N}Y^{-i-N})$$
$$s(X, Y) = \sum_{i=1}^N(u_i(Y)X^{-i} + v_i(Y)X^i + w_i(Y)X^{i+N})$$
$$s'(X, Y) = Y^Ns(X, Y) - \sum_{i=1}^N(Y^i + Y^{-i})X^{i+N}$$
$$t(X, Y) = r(X, 1)(r(X, Y) + s'(X, Y)) - Y^Nk(Y) \tag{8}$$

Note that the constant term of $t(X, Y)$ in terms of $X$, that is, the coefficient of $X^0$, is exactly the left-hand side of Equation 7. So the protocol design is to show that, Equation 8 still holds after removing the constant term from the left-hand side.

#### Single Sub-Circuit Proving

This part is still mostly Sonic Arithmetization with (modified) IPA. Some ideas come from [Halo](https://eprint.iacr.org/2019/1021.pdf) but we do not use the amortizing part or cycles of curves part. We stick to the `secp256k1` curve. Also as we hard-coded some circuit constants in the verifier side, verifier could compute these values on its own. 

Prover commits to $r(X, 1)$ with blinder $\delta_R$ for circuit $C$. We follow [Halo paper](https://eprint.iacr.org/2019/1021.pdf) to scale it with $X^{3N-1}$ to ensure $r(X, 1)$ is at most degreen $N$.
$$\delta_R \overset{\$}{\larr} \mathbb{Z}_p,\quad R \larr \text{Commit}(r(X, 1)X^{3N-1}; \delta_R) $$

Verifier responds with a challenge value $y \in \mathbb{Z}_p$. In a non-interactive setting, $y$ should be generated by hashing the past transcriptions up to and including the commitment value $R$. We ignore this notion in later sections.

Now let $t_{lo}(X, y), t_0(y), t_{hi}(X, y)$ be the part of $t(X, y)$ with negative exponent, constant term, and with positive exponent, for X, that is,
$$t(X, y) = t_{lo}(X, y)X^{-d} + t_0(y) + t_{hi}(X, y)X$$

Prover commits to $t_{lo}$ and $t_{hi}$.
$$\delta_{T_{lo}} \overset{\$}{\larr} \mathbb{Z}_p,\quad T_{lo} \larr \text{Commit}(t_{lo}(X, y); \delta_{T_{lo}}) $$
$$\delta_{T_{hi}} \overset{\$}{\larr} \mathbb{Z}_p,\quad T_{hi} \larr \text{Commit}(t_{hi}(X, y); \delta_{T_{hi}}) $$

Verifier responds with a challenge value $z \in \mathbb{Z}_p$.

Prover now needs to open a few commitments, and provides open to values and proofs.
$$(e = r(z, 1), proof_a) \larr \text{Open}_\text{IPA}(R, z, r(X, 1), \delta_R)$$
$$(f = r(z, y), proof_b) \larr \text{Open}_\text{IPA}(R, yz, r(X, 1), \delta_R)$$
$$(t_1 = t_{lo}(z, y), proof_{t_{lo}}) \larr \text{Open}_\text{IPA}(T_{lo}, z, t_{lo}(X, y), \delta_{T_{lo}})$$
$$(t_2 = t_{hi}(z, y), proof_{t_{hi}}) \larr \text{Open}_\text{IPA}(T_{hi}, z, t_{hi}(X, y), \delta_{T_{hi}})$$

Verifier need to verify all the above openings, compute $s = s(z, y)$, $s'$ from $s$, $k = k(y)$, then check if
$$t_1 + t_2 \overset{?}{=} e(f + s') - k \tag{8}$$

Passing Identity Test 8 means that $t_0(y) = 0$ with overwhelming probability. If all checks pass, Verifier returns 1, otherwise 0.

*Remark* The computation of $s$ does not involve confidential witness data $\textbf{a}, \textbf{b}$ or $\textbf{c}$, so Verifier could compute on its own. Note that although $\textbf{U}, \textbf{V}, \textbf{W}$ are sparse for each sub-circuit, sparseness no longer holds after aggregating many sub-circuits. The size would exdeed the bound for opening $R$, $T_{lo}$ or $T_{hi}$. Opening a commitment to $s(X, y)$ is therefore not viable for aggregated proving. Direct computation in $\mathbb{Z}_p$ should be much faster than verifying opening with IPA, which features multi-scalar multiplication of linear size in $\mathbb{G}$. That's the reason we don't bother committing to it even for single sub-circuit. We will discuss the optimization for computing $s$ in later sections.

##### Batched Opening for Sub-Circuit

The opening of $R$ at $z$, $yz$, of $T_{lo}$ and $T_{hi}$ at $z$, could be batched together with the batching techniques we have described in the last section.

Specfically, $R$ could be opened to $e \in \mathbb{Z}_p$ and $f \in \mathbb{Z}_p$ at $z$ and $yz$ given random generator $U_1$ and $U_2$. We have
$$P_R = R + [e]U_1 + [f]U_2 = \langle\textbf{r}, \textbf{G} \rangle + [\delta_R]H + [\langle \textbf{r}, \textbf{z} \rangle]U_1 + [\langle \textbf{r}, \textbf{zy} \rangle]U_2$$
where $\textbf{r}$ denotes the coefficiencies for $r(X, 1)$.

To open $T_{lo}$ and $T_{hi}$ at $z$ in a batch along with $R$, suppose $\beta$ is the challenge value, we should have
$$P = R + [\beta]T_{lo} + [\beta^2]T_{hi} + [e + \beta \cdot t_{lo} + \beta^2 \cdot t_{hi}]U_1 + [f]U_2$$
$$\quad = \langle\textbf{r} + \beta \cdot \textbf{t}_\textbf{lo} + \beta^2 \cdot \textbf{t}_\textbf{hi}, \textbf{G} \rangle + [\delta_R + \beta \cdot \delta_{lo} + \beta^2 \cdot \delta_{hi}]H $$
$$\quad \quad + [\langle \textbf{r} + \beta \cdot \textbf{t}_\textbf{lo} + \beta^2 \cdot \textbf{t}_\textbf{hi}, \textbf{z} \rangle]U_1 + [\langle \textbf{r}, \textbf{zy} \rangle]U_2 \tag{9}$$

