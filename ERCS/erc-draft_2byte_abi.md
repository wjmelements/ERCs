---
title: 2byte Batch ABI
description: Sequential batch specification minimizing calldata for modular batchers
author: William Morriss (@wjmelements)
discussions-to: <URL>
status: Draft
type: Standards Track
category: Interface
created: <date created on, in ISO 8601 (yyyy-mm-dd) format>
---

## Abstract

By concatenating subprograms, a modular smart contract architecture emerges suitable for personal accounts.

## Motivation

The only standardized ABI at this time is the 4byte ABI designed by Solidity and adopted by other compilers like Vyper.
This specification uses substantially less calldata than alternative batching designs implementing the 4byte ABI, such as [ERC-4337](./erc-4337.md).

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

The calldata to a 2byte batcher is a concatenation of subcommands each starting with a two-byte prefix selecting a `JUMPDEST` for execution.
The batcher MAY execute these subcommands sequentially, possibly reverting early for any reason.

### Subprograms

A subprogram:
* MUST start with a `JUMPDEST`.
* MUST finish with a conditional or unconditional jump to the next subprogram.
* SHOULD modify the calldata indices.
* SHOULD NOT pollute the stack with unused remnants.

### Common State

Some state MAY be common to all subprograms.

It is RECOMMENDED to preserve such state in the base of the stack.

The batcher MAY also use temporary storage for state common to all subprograms.

#### Calldata Index

A calldata index maintains the batcher's progress in the workload.

It is RECOMMENDED to maintain only one calldata index, at the top of the common stack.

### Authenticate-Last

A batcher MAY authenticate-last by prohibiting subprograms that would allow an early `STOP`, `RETURN`, or `SELFDESTRUCT`, except after final authentication.

Subprograms requiring additional authentication SHOULD flag this for later verification

### Code Header 

The batcher code SHOULD begin:

```
36600557005b5f5f3560f01c565b595ffd
```

Thus the standard `revert` index is `0x0d`.

## Rationale

This batch architecture uses two-byte prefixes because the current `CODESIZE` limit fits in two bytes.
If this limit were larger, larger prefixes can be used, but this also brings larger calldata.

jaredfromsubway.eth developed a similar batcher that pads each subprogram to 256 bytes in order to use a 1byte prefix. 
This 1byte ABI can be more efficient depending on the cost of additional code size from the padding compared to the number of expected executions.

While most subprograms should increase the calldata index by two plus the length of their parameters, allowing subprograms to rollback the calldata index allows looping groups of subprograms through a list of parameters while reusing calldata.

While most subprograms should not change the number of calldata indices, allowing multiple calldata indices can facilitate internal functions.

Authenticate-last saves gas in the case where the batch fails for reasons other than authentication, while allowing multitudes of authentication routes, for no additional execution overhead.

The modular design allows for easy development.
Subprograms are shareable plug-and-play systems and can be registered on-chain for detection, security audits, compatibility notices, and wallet support.
The modular design also facilitates upgrades.
Subprograms updating the batcher can safely splice subprograms in and out without breaking other functionalities. 
This update is easiest with `SETCODE` but also works for `DELEGATECALL` proxies.

The standardized header makes for easy identification of 2byte batchers.
Wallets supporting the 2byte ABI can scan subprograms by splitting the remainder on `JUMPDEST` and looking them up in registries.
Wallets can then support matching subprograms with named and typed parameters.

## Backwards Compatibility

The 2byte Batch ABI is not backwards compatible with the 4byte ABI, but contracts implementing these designs can still interact with each other.

## Reference Implementation

This is a 2byte batcher implementation for owner `0x4a6f6B9fF1fc974096f9063a45Fd12bD5B928AD1` with subprograms `logzero`, `callwithvalue`, `callwithoutvalue`, and `authstopowner`.
It practices "Authenticate-Last" and maintains a calldata index at the top of the common stack. 
```
JUMPI(init, CALLDATASIZE)
STOP

init:
0
JUMP(SHR(240, CALLDATALOAD(0)))

revert:
REVERT(0, MSIZE)

logzero:
// [dataSize1, data...]
2 ADD
BYTE(0, CALLDATALOAD(DUP1)) // dataSize
CALLDATACOPY(0, DUP3, DUP1)
LOG0(0, DUP1)
ADD
JUMP(SHR(240, CALLDATALOAD(DUP1)))

callwithvalue:
// [inSize2, address20, value10, calldata]
34 ADD
CALLDATALOAD(SUB(DUP2, 32))
0 // outSize
0 // outStart
SHR(240, DUP3) /// inSize
CALLDATACOPY(0, DUP6, DUP1)
0 // inStart
AND(0xffffffffffffffffffff, DUP5) // value
ADD(DUP8, DUP3) SWAP6 80 SHR // to
GAS
JUMPI(SHR(240, CALLDATALOAD(DUP2)), CALL)
REVERT(0, MSIZE)

callwithoutvalue:
// [inSize2, address20, calldata]
24 ADD
CALLDATALOAD(SUB(DUP2, 22))
0 // outSize
0 // outStart
SHR(240, DUP3) // inSize
CALLDATACOPY(0, DUP6, DUP1)
0 // inStart
0 // value
ADD(DUP8, DUP3) SWAP6 80 SHR // to
GAS
JUMPI(SHR(240, CALLDATALOAD(DUP2)), CALL)
REVERT(0, MSIZE)

authstopowner:
JUMPI(revert, XOR(0x4a6f6B9fF1fc974096f9063a45Fd12bD5B928AD1, CALLER))
STOP
```

## Security Considerations

`JUMPDEST`-validity regulates valid subprogram starts.

Subprograms not designed to be authentication-stops should not contain `STOP`, `RETURN`, or `SELFDESTRUCT`.
Any such stopping opcodes accessible after a `JUMPDEST` can commit all previous activity with no further authentication.

Batcher code SHOULD NOT begin with `JUMPDEST` because a subprogram consuming the end of calldata should abort execution by jumping to the invalid destination `0x0000`.

A registry of subprograms can help with their reputation and allow wallets to warn users of unaudited subprograms.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
