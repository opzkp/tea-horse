# The Tea/Horse Proving System

by Weiji Guo from Lightec Labs

*Note: by the time of writing, this document has NOT been reviewed. It may contain mistakes of various kinds. Also most claims are NOT backed by benchmarks. This is so far a live document rather than a paper draft intended to be published. We will address reviewing and benchmarking as we move on.*

We are developing the Tea/Horse proving system as the underlying scheme for the proposed OP_ZKP soft-fork upgrade to Bitcoin.

### Introduction

The Tea/Horse proving system consists of three major parts:

- Inner-Product Argument (IPA), thus the (EC)DLP hardness assumption only, and the secp256k1 curve only.
- Aggregated Proving to effectively reduce the symptotical Verifier complexity to sub-linear.
- Single circuit for a recursive verifier, to address the linear verification key issue.

The name Tea/Horse comes from an ancient confidential transaction style, in which people traded tea for horse or vice versa, (perhaps sometimes) keeping the price/bidding information within sleeves, exchanged during hand-shaking. Below picture was taken in Songpan where the author was there to attend a trail running event early July. He came up with the idea of aggregated proving on his flight going there.

![confidential transaction trading tea vs horse](img/th.jpeg)

### Technical Requirements and Key Design

As had been explained in [an email to the bitcoin developer mailing list](https://groups.google.com/g/bitcoindev/c/YEXcac4FMGc), we are considering **Inner-Product Argument**. We could achieve succinct (sub-linear) verifier with **aggregated proving**, but are left with one open issue. That is, the verification key is too large for on-chain deployment.

Searching for a commitment scheme to address this open issue does not land a definitive answer. Instead, it seems quite the opposite, that we might not be able to find such a scheme. Circuit constants are essentially matrixes of size O(N^2), however sparse. And as pointed out before, these matrixes may not be sparse any more after adopting aggregated proving.

Luckily there is a third option besides (1)committing to the circuit constants, or (2)turning to [Dory](https://eprint.iacr.org/2020/1274.pdf). That is, **recursive verification**. The idea is straightforward. Instead of verifying proofs from individual circuits directly, the OP_ZKP implementation has a verifier to verify proofs from one special circuit (the recursive circuit), which in turn recursively verifies proofs for business logic related circuits (application circuits).

The advantages are two fold:

(1) There is no need to deploy verification keys on-chain. The constants (and scheme-related public parameters) of the recursive verifier could be hard-coded in the Bitcoin client, taking up no block space. As for all application circuits, their circuit constants could be provided as witness to the recursive prover, thus hidden from the recursive verifier.

(2) It is possible to upgrade the recursive prover, the recursive verifier or the circuit without interrupting the applications based on OP_ZKP, in case of critical security patches. Still, it is each application circuit's responsibility to take care of their own security and efficiency.

