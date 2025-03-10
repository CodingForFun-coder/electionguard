# ElectionGuard Components

This document describes the four principal components of ElectionGuard.

1. Baseline Parameters – These are general parameters that are standard in every election. An alternate means for generating parameters is described, but the burden of verifying an election is increased if alternate parameters are used because a verifier would need to verify the proper construction of any alternate parameters.

2. Key Generation – Prior to each individual election, guardians must generate individual public-private key pairs and exchange shares of private keys to enable completion of an election even if some guardians become unavailable. Although it is preferred to generate new keys for each election, it is permissible to use the same keys for multiple elections so long as the set of guardians remains the same. A complete new set of keys must be generated if even a single guardian is replaced.

3. Ballot Encryption – While encrypting the contents of a ballot is a relatively simple 
operation, most of the work of ElectionGuard is the process of creating externallyverifiable artifacts to prove that each encrypted ballot is well-formed (i.e., its decryption is a legitimate ballot without overvotes or improper values).

4. Verifiable Decryption – At the conclusion of each election, guardians use their private keys to produce election tallies together with verifiable artifacts that prove that the tallies are correct.

## Notation

In the remainder of this specification, the following notation will be used.

• $ℤ = \{… , −3, −2, −1, 0, 1, 2, 3, … \}$ is the set of integers.

• $ℤ_n = \{0, 1, 2, … , n − 1\}$ is the additive group of the integers modulo $p$.

• $ℤ_n^∗$ is the multiplicative subgroup of $ℤ_n$. When $p$ is a prime, $ℤ_p^* = \{1, 2, 3, … , p − 1\}.$

• $ℤ_p^r$ is the set of $r^{th}$ -residues in $ℤ_p^*$. Formally, $ℤ_p^r = \{y \in ℤ_p^*$ for which $\exists x \in ℤ_p^∗$ such that $y = x^r \bmod p\}$. When $p$ is a prime for which $p − 1 = qr$ with $q$ a prime that is not a divisor of integer $r$, then $ℤ_p^r$ is an order $q$ cyclic subgroup of $ℤ_p^∗$ and for each $y \in ℤ_p^∗, y \in ℤ_p^r$
if and only if $y^q \bmod p = 1$.

• $x \equiv_n y$ is the predicate that is true if and only if $x \bmod n = y \bmod n$.

• The function $H()$ shall be used to designate the SHA-256 hash function (as defined in NIST PUB FIPS 180-4[^1]).

• In general, the variable pairs $(\alpha, \beta)$, $(a, b)$, and $(A, B)$ will be used to denote encryptions. Specifically, $(\alpha, \beta)$ will be used to designate encryptions of votes (always an encryption of a zero or a one), $(A, B)$ will be used to denote aggregations of encryptions – which may be encryptions of larger values, and $(a, b)$ will be used to denote encryption commitments used to prove properties of other encryptions.

## Encryption of Votes

Encryption of votes in ElectionGuard is performed using an exponential form of the ElGamal cryptosystem.[^2] Primes $p$ and $q$ are publicly fixed such that $q$ is not a divisor of $r = \frac {p−1} {q}$. A generator $g$ of the order $q$ subgroup $ℤ_p^r$ is also fixed. (Any $g = x^r \bmod p$ for which $x \in ℤ_p^*$ suffices so long as $g \neq 1$.)

A public-private key pair can be chosen by selecting a random $s \in ℤ_q$ as a private key and publishing $K = g^s \bmod p$ as a public key.[^3]

A message $M \in ℤ_p^r$ is then encrypted by selecting a random nonce $R \in ℤ_q$ and forming the pair $(\alpha, \beta) = (g^R \bmod p,g^M ⋅ K^R \bmod p)$. An encryption $(\alpha, \beta)$ can be decrypted by the holder of the secret $s$ as

$\frac \beta {\alpha^s} \bmod p =\frac {g^M ⋅ K^R} {(g^R)^s} \bmod p = \frac {g^M ⋅ (g^s)^R} {(g^R)^s} \bmod p = \frac {g^M ⋅ g^{Rs}} {g^{Rs}} \bmod p = g^M \bmod p.$

The value of $M$ can be computed from $g^M \bmod p$ as long as the message $M$ is limited to a small, known set of options.[^4]

Only two possible messages are encrypted in this way by *ElectionGuard*. An encryption of one is used to indicate that an option is selected, and an encryption of zero is used to indicate that an option is *not* selected.

## Homomorphic Properties

A fundamental quality of the exponential form of ElGamal described above is its additively homomorphic property. If two messages $M_1$ and $M_2$ are respectively encrypted as $(A_1,B_1) =(g^{R_1} \bmod p, g^{M_1} ⋅ K^{R_1} \bmod p)$ and $(A_2,B_2) = (g^{R_2} \bmod p, g^{M_2} ⋅ K^{R_2} \bmod p)$, then the component-wise product $(A, B) = (A_1A_2 \bmod p,B_1B_2 \bmod p) = (g^{R_1+R_2} \bmod p, g^{M_1+M_2}⋅K^{R_1+R_2} \bmod p)$ is an encryption of the sum $M_1 + M_2$. (There is an implicit assumption here 
that $(M_1 + M_2) < q$ which is easily satisfied when $M_1$ and $M_2$ are both small. If $(R_1 + R_2) ≥ q, (R_1 + R_2) \bmod q$ may be substituted without changing the equation since $g^q \bmod p = 1.)$

This additively homomorphic property is used in two important ways in ElectionGuard. First, all of the encryptions of a single option across ballots can be multiplied to form an encryption of the sum of the individual values. Since the individual values are one on ballots that select that option and zero otherwise, the sum is the tally of votes for that option and the product of the individual encryptions is an encryption of the tally.
The other use is to sum all of the selections made in a single contest on a single ballot. After demonstrating that each option is an encryption of either zero or one, the product of the encryptions indicates the number of options that are encryptions of one, and this can be used to show that no more ones than permitted are among the encrypted options – i.e., that no more options were selected than permitted.

However, as will be described below, it is possible for a holder of a nonce $R$ to prove to a third party that a pair $(\alpha, \beta)$ is an encryption of $M$ without revealing the nonce $R$ and without access to the secret $s$.

## Non-Interactive Zero-Knowledge (NIZK) Proofs

ElectionGuard provides numerous proofs about encryption keys, encrypted ballots, and election tallies using the following four techniques.

1. A Schnorr proof[^5] allows the holder of an ElGamal secret key $s$ to interactively prove possession of $s$ without revealing $s$.

2. A Chaum-Pedersen proof[^6] allows an ElGamal encryption to be interactively proven to decrypt to a particular value without revealing the nonce used for encryption or the secret decryption key $s$. (This proof can be constructed with access to either the nonce used for encryption or the secret decryption key.)

3. The Cramer-Damgård-Schoenmakers technique[^7] enables a disjunction to be interactively proven without revealing which disjunct is true.

4. The Fiat-Shamir heuristic[^8] allows interactive proofs to be converted into noninteractive proofs.

Using a combination of the above techniques, it is possible for ElectionGuard to demonstrate that keys are properly chosen, that ballots are properly formed, and that decryptions match claimed values.

## Threshold Encryption

Threshold ElGamal encryption is used for encryption of ballots and other data. This form of encryption makes it very easy to combine individual guardian public keys into a single public key. It also offers a homomorphic property that allows individual encrypted votes to be combined to form encrypted tallies.

The guardians of an election will each generate a public-private key pair. The public keys will then be combined (as described in the following section) into a single election public key which is used to encrypt all selections made by voters in the election.

Ideally, at the conclusion of the election, each guardian will use its private key to form a verifiable partial decryption of each tally. These partial decryptions will then be combined to form full verifiable decryptions of the election tallies.

To accommodate the possibility that one or more of the guardians will not be available at the conclusion of the election to form their partial decryptions, the guardians will cryptographically share [^9] their private keys amongst each other during key generation in a manner to be detailed in the next section. A pre-determined threshold quorum value $(k)$ out of the $(n)$ guardians will be necessary to produce a full decryption.

[^1]: NIST (2015) Secure Hash Standard (SHS). In: FIPS 180-4. https://csrc.nist.gov/publications/detail/fips/180/4/final

[^2]: ElGamal T. (1985) A Public Key Cryptosystem and a Signature Scheme Based on Discrete Logarithms. In: Blakley G.R., Chaum D. (eds) Advances in Cryptology. CRYPTO 1984. Lecture Notes in Computer Science, vol 196. Springer, Berlin, Heidelberg. https://link.springer.com/content/pdf/10.1007/3-540-39568-7_2.pdf

[^3]: As will be seen below, the actual public key used to encrypt votes will be a combination of separately-generated public keys. So no entity will ever be in possession of a private key that can be used to decrypt votes.

[^4]: The simplest way to compute $M$ from $g^M \bmod p$ is an exhaustive search through possible values of $M$. Alternatively, a table of pairing each possible value of $g^M \bmod p$ can be pre-computed. A final option which can accommodate a larger space of possible values for $M$ is to use Shanks’s baby-step giant-step method as described in the 1971 paper “Class Number, a Theory of Factorization and Genera,” Proceedings of Symposium in Pure Mathematics, Vol. 20, American Mathematical Society, Providence, 1971, pp. 415-440.

[^5]: Schnorr C.P. (1990) Efficient Identification and Signatures for Smart Cards. In: Brassard G. (eds) Advances in Cryptology — CRYPTO’ 89 Proceedings. CRYPTO 1989. Lecture Notes in Computer Science, vol 435. Springer, New York, NY. https://link.springer.com/content/pdf/10.1007%2F0-387-34805-0_22.pdf

[^6]: Chaum D., Pedersen T.P. (1993) Wallet Databases with Observers. In: Brickell E.F. (eds) Advances in Cryptology — CRYPTO’ 92. CRYPTO 1992. Lecture Notes in Computer Science, vol 740. Springer, Berlin, Heidelberg. https://link.springer.com/content/pdf/10.1007%2F3-540-48071-4_7.pdf

[^7]: Cramer R., Damgård I., Schoenmakers B. (1994) Proofs of Partial Knowledge and Simplified Design of Witness Hiding Protocols. In: Desmedt Y.G. (eds) Advances in Cryptology — CRYPTO ’94. CRYPTO 1994. Lecture Notes in Computer Science, vol 839. Springer, Berlin, Heidelberg. https://link.springer.com/content/pdf/10.1007%2F3-540-48658-5_19.pdf

[^8]: Fiat A., Shamir A. (1987) How To Prove Yourself: Practical Solutions to Identification and Signature Problems. In: Odlyzko A.M. (eds) Advances in Cryptology — CRYPTO’ 86. CRYPTO 1986. Lecture Notes in Computer Science, vol 263. Springer, Berlin, Heidelberg. https://link.springer.com/content/pdf/10.1007%2F3-
540-47721-7_12.pdf

[^9]:  Shamir A. How to Share a Secret. (1979) Communications of the ACM.
