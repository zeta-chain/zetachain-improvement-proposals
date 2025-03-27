---
zip: 9
title: Threshold BLS Signature for TSS on Connected Chains
status: LastCall
author: brewmaster012 
created: 2025-02-24
---


### Simple Summary
ECDSA TSS is slow and requires synchronization of signers to keysign. 
This ZIP proposes alternative TSS based on BLS signature scheme
that promises orders of magnitute higher keysign throughput and lower latency, 
no signer synchrony requirement, straight-forward on-chain accounting
of keysign participation, and much simpler signer set change. 


### Abstract and Motivation
ZetaChain relies on accounts on each of its connected chains to authenticate
actions commanded by ZetaChain. To reduce single point of failure and improve
trust via decentralization, the secret keys of external accounts are stored and
distributed among a set of special ZetaChain validators called Signers. 

Right now the Signers collectively possess a single ECDSA (secp256k1 curve)
using variant of GG18/GG20 ECDSA TSS scheme. While its compatibility is
excellent (supported natively on Ethereum and Bitcoin, and SUI, and via
contracts on Solana and TON), it's lacking in the following important aspects: 

1. GG20 ECDSA is very computational expensive (e.g. keysign involves several
   rounds of ceremonies, each one might involve quite a few zk-proofs including
   some expensive ones like range proofs to ensure secure multi-party computation). 
   The result is that keysign latency is a few seconds and throughput is severely
   limited (quite a bit less than 1 sig/s on decent cloud VMs). 
2. GG20 ECDSA keysign and keygen are synchronous (as name suggests, ceremonies)
   requiring all signers to synchronize. This requires all signers to schedule keysign
   at more or less the same time, and each round comes with a timeout. Not sufficiently
   synchronized keysign not only causes failure; it consumes more resources like
   p2p streams for longer, and might cause cascading failures. This poses some
3. GG20 ECDSA keysign failure attribution is hard and expensive to pinpoint and reach
   consensus on, which hinders on-chain automatic incentives required for
   permisssionless participation. 
   
This motivates exploring other TSS schemes without those drawbacks.
The most suitable cryptographic solution is threshold BLS signature
which admits a much simpler MPC scheme that does not require any
zero-knowledge proof, nor any rounds of ceremonies. It's likely 100x
faster to keysign than GG20 ECDSA both in principle and in
implementation. The drawback is its compatibility, which can be
addressed via contract verification on all current and expected
external chains other than Bitcoin. 

## Specification
### Key Gen
Suppose there are n signers. Set threshold t = 2/3 *n. 
The Signers perform a distriburted key generation (DKG) of 
threshold secret sharing, essentially doing n copies of Verified
Secret Sharing (VSS) dealt by every signer. 

This is a one round process, no synchrony required. 
Communication channel is a broadcast channel signed by the signers. 
An obvious choice is the ZetaChain state machine as the reliable broadcast
channel. 

### Key Sign
Each signer uses its own keyshare to do a BLS keysign on the hashed transaction.
Each signer then broadcast its signature to other signers, and any one of the signer
can recover the valid signature from more t+1 signatures. 

This is a one round process, no synchrony required. 
Communication channel is a broadcast channel signed by the signers. 
An obvious choice is the ZetaChain state machine as the reliable broadcast
channel. 

### Reshare
When a composition of signers set changes they must regenerate a new
keyshare and get rid of the old ones. It would be advantageous to re-share
the collective "secret" into a new set of keyshared (of course doing so
in distributed way without any single point in time where the secret is
in one party's hand).  Reshare maintains the same collective secret key and therefore
the same public key and address on external chains, reducing risks in potential
migrations which is necessary for a new keygen. 

### BLS Curves
The threshold BLS signature works the same regardless of which EC curve
it's operating on. Currently there are two popular choices, BN254 and BLS12-381
curve. The former is older with about 100 bits security entropy while the latter
is 128bits. The former is the one that zk-SNARKS Groth16 scheme often uses. 

On EVM chains (except Ethereum) the older BN254 (also called alt_bn128) curve must be used as
it's the only one that's currently efficiently supported via precompiles. 

New Ethereum upgrade Pectra (expected on mainnet on Mar 2025) includes precompiles
support for BLS12-381 and its standard hash to curve. Other EVM chains might follow. 

On Solana the BN254 curve must be used as it's efficiently supported via syscall. 

On SUI the [BLS12-381 curve] must be used as it's efficiently supported via
native functions. 

On TON network, [BLS12-381 curve](https://docs.ton.org/v3/documentation/tvm/changelog/tvm-upgrade-2023-07#bls12-381) must be used.

### Hash To Curve
In BLS signature one needs to hash a transaaction/message into a point on the G1
EC curve in order to verify. 

For BN254 a [reasonable choice](https://eips.ethereum.org/assets/eip-3068/weilsigs.pdf)
is to hash the message using Keccak256 for x, 
and find a y that satisfies the curve equation y^2 = x^3 +3. If no y exists, then
increment x and try again. There is 50% chance each time to succeed. Practically
20 max tries would be enough to cap the gas cost. The average tries should be 2. 
There will be a very small chance that hash would fail for certain message (20 tries
does not yield successful point), then zetaclient should be aware of this and change
the message without affecting its semantics (i.e. adding CALLDATA as memo to EVM tx, 
adding a NOOP to the contract call or a NOOP transaction to Solana etc). 


For BLS12-381 there is IETF RFC 9380
[BLS12381G1_XMD:SHA-256_SSWU_RO_](https://datatracker.ietf.org/doc/html/draft-irtf-cfrg-hash-to-curve-05#section-8.9)
that is supported everywehere where the BLS12-381 curve is supported. 
The hash function will be provided as precompiles on as part of EIP-2537. 


### Signature in G1
BLS signature can be on either the G1 or G2, resulting in two flavors: 

- Signature on G1, Public Key on G2
- Signature on G2, Public Key on G1

As G1 is smaller than G2, and we store and process signatures much more than public
key, the first scheme (signature on G1) is recommended in this ZIP. 

## Rationale 
### Why tBLS? Any other choices? 
BLS is widely supported on all smart contract chains. Its threshold form
is the simplest MPC without any complicated zk-proofs. 


### Verification Costs
On EVM, a verification could cost around 110K gas including hash and pairing on BN254 curve. 
On BLS120381 (part of Ethereum Pectra upgrade) it remains to be tested but could be
smaller gas cost. 

On Solana, around 100K CU (PoC) or less (can be further optimized) for verification. There is
room for further reduction but 100K CU is quite acceptable. 

On SUI, cost is very low as the whole verificaiton is implemented natively. 

On TON, likely very low as the whole verification is implemented natively. 

### Communication Channel: Broadcast channel with ZetaChain state machine
The Distributed KeyGen (DKG) and KeySign protocol requires communication between signers; 
Some kind of broadcast channel is required for DKG but not necessary for KeySign. 
However for possible on-chain accoutability of signers broadcast channel via ZetaChain
state machine would be idea, as all actions are recorded on-chain and can be automatically
accounted and incentivized. On the other hand, having KeySign on ZetaChain means 
each outbound transaction will require at least #signers amount of transactions in ZetaChain,
therefore putting an upper limit on the outbound TPS to peak ZetaChain TPS/#signers. 

An alternative to using the ZetaChain state machine as the broadcast channel is
a custom p2p PubSub topic. The advantage is no overhead to the main ZeatChain network; 
drawback is PubSub is best-effort and unreliable broadcast channel (no guarantee of
delivery, no ordering, and no consensus that everyone got broadcasted the same message). 

## Backward/Forward Compatibility
This is not backward compatible to ECDSA TSS. 

For forward compatibility, it's unlikely BLS curve operations will be deprecated 
any time soon. 

On Ethereum, [EIP-2537](https://eips.ethereum.org/EIPS/eip-2537) adds BLS12-381
to precompiles, likely to be alive soon in 2025 as part of Pectra upgrade. The code
has been merged in [Geth PR](https://github.com/ethereum/go-ethereum/pull/30978).
It's likely some other EVM chains will adopt this EIP over time.

As Ethereum deploys BLS12-381 precompiles I'd expect all other chains to support
it some time including Solana. 


## Test Cases

## Proof-of-Concept
On EVM BN254 curve:  https://github.com/brewmaster012/bls-solidity

On SUI BLS12-381 curve: https://github.com/zeta-chain/protocol-contracts-sui/tree/bls-poc

On Solana BN254 curve: https://github.com/brewmaster012/solana-bls-poc

On TON BLS12-381 curve: TBD

On zetaclient side, the BN254 with the hash function patch:  https://github.com/brewmaster012/kyber
Threshold BLS is also part of Kyber v4. 


## Implementation
