# CHIP-2023-07-Composite-Arithmetic-Opcodes

        Title: Composite Math Opcodes
        Type: Standards
        Layer: Consensus
        Owners: Mathieu Geukens & bitcoincashautist 
        Status: Draft
        Initial Publication Date: 2023-07-05
        Latest Revision Date: 2023-07-07

## Summary

This proposal adds four new math opcodes to the Bitcoin Cash Virtual Machine (BCH VM), `op_muldiv`, `op_mulmod`, `op_adddiv` and `op_addmod`.

`op_muldiv` greatly simplifies contracts using 64-bit integer multiplication in intermediate calculations.
These calculations currently fail on overflow so contract authors need to either restrict input ranges or use clever math workarounds which sacrifice precision.

Together, the set of composite opcodes allows contracts to emulate higher order math (int127) easily using 64-bit integers as demonstrated 
by the included example CashScript Math library. Allowing contract authors to design their contracts for the precision needed for their specific usecase.

## Deployment

Deployment of this specification is proposed for the May 2024 upgrade.

## Motivation

Bitcoin Cash has upgraded to 64-bit integers and now with the advent of CashTokens the BCH VM enables complex applications,
this poses the need both for higher-order math and for allowing overflows in intermediate calculations.

Now, shortly after the CashTokens upgrade there are already multiple different Constant ProductMarket Maker (CPMM) contracts live on the network 
which carefully have to consider strategies for multiplication overflows.
The composite arithmetic opcode `op_muldiv` allows the intermediate calculation to overflow and hence simplifies the strategies dealing with overflows,
leading to simpler, more efficient and more capable smart contracts.

Moreover, the introduction of composite arithmetic opcodes for multiplication and addition enables convenient emulation of higher precision math in BCH contracts.
This way contract authors can design their contract for the precision needed for their specific usecase.

## Benefits

By introducing these opcodes, the code for advanced contracts such as AMMs can be improved in multiple dimensions and 
more advanced and precise contracts using higher-order math emulation are enabled.

### Simpler, More Efficient & More Capable Contracts

There are two overflow strategies for contracts using mutiplication in an intermediate step: 
- Restrict the range of the coefficients so they can be guaranteed not to overflow 
- Use modulo math to approximate the end result of the calculation

In the former case, the introduction of `op_muldiv` would allow contract authors to expand the ranges of allowed coefficients easily.
In the latter  case, the introduction of `op_muldiv` would allow contract authors to simplify their contract by getting rid of the 
modulo math workaround and instead use the native functionality. This would make the contract simpler (less error-prone) and more efficient
both in terms of opcodes and bytesize.

[Cauldron DEX](https://www.cauldron.quest/) is an example of the first strategy and [Fex.cash](https://fex.cash/) of the second one.
Both projects would benefit from having the option to utilize `op_muldiv` in a future version.
Even UniSwap V3 makes heavy use of a [`muldiv` helperfunction](https://docs.uniswap.org/contracts/v3/reference/core/libraries/FullMath) and has [taken initiative](https://ethereum-magicians.org/t/create-a-new-opcode-for-muldiv/8574) to get an `muldiv` opcode added to the EVM.

### Higher Precision Math

With `op_muldiv` and `op_mulmod` int127 multiplication can be easily emulated with full precision by using two 64-bit integers
to represent a int127 (as shown [here](https://medium.com/wicketh/mathemagic-full-multiply-27650fec525d)).
The addition and subtraction of two int127 numbers is made easy by the `op_adddiv` and `op_addmod` opcodes, 
however for division [CHIP-2021-05-loops: Bounded Looping Operations](https://github.com/bitjson/bch-loops) is required in addition to `op_muldiv` 
to make it practical (as shown [here](https://medium.com/wicketh/mathemagic-512-bit-division-in-solidity-afa55870a65)).

The four new composite arithmetic opcodes serve as primitives for building user-space libraries that enable higher order (int127) math.
CashScript is investigating [enabling support for libraries/macros](https://github.com/CashScript/cashscript/issues/153) 
so a user-space library for int127 math could be built and utilized by other contracts.
In this CHIP an example `Math.cash` CashScript library is added to demonstrate the simplicity of int127 math with the introduction of the new opcodes.

### Easy overflow check

If a contract has conditional logic for when a calculations would overflow, this condition can be easily check using `op_muldiv` and `op_adddiv` by using the maximum integer as divisor. Then the result return a number `!=0` on overflow and `0` otherwise. 

## Costs & Risk Mitigation

The following costs and risks have been assessed.

### Modification to Transaction Validation

Modifications to VM limits have the potential to increase worst-case transaction validation costs and expose VM implementations to Denial of Service (DOS) attacks.

**Mitigations**: A simple arithmetic sequence such as multiplying-dividing or multiplying-modulo should have no impact on worst-case transaction validation costs 
and the DOS-attack surface.  

### Node Upgrade Costs
 
This proposal affects consensus – all fully-validating node software must implement these VM changes to remain in consensus.

These VM changes are backwards-compatible: all past and currently-possible transactions remain valid under these new rules.

### Ecosystem Upgrade Costs

Because this proposal only affects internals of the VM, standard wallets, block explorers, and other services will not require software modifications for these changes. Only software which offers emulation of VM evaluation (e.g. [Bitauth IDE](https://github.com/bitauth/bitauth-ide)) will be required to upgrade.

Wallets and other services may also upgrade to add support for new contracts which will become possible after deployment of this proposal.

## Technical Specification

This specification proposes to use four adjacent codepoints available in the `OP_UNKNOWN` range for the set of composite arithmetic opcodes:

|Word|Value|Hex|Input|Output|Description|
| --- | --- | --- | --- | --- | --- |
|OP_MULDIV|218|0xda|a b c|out|*a* is first multiplied by *b* then divided by *c* and the quotient of the division is returned. The intermediate product CAN be in the INT128 range, while the quotient MUST be in the INT64 range. Sign is preserved and *c* MUST be a non-zero value.|
|OP_MULMOD|219|0xdb|a b c|out|*a* is first multiplied by *b* then divided by *c* and the modulus of the division is returned. The intermediate product CAN be in the INT128 range, while the modulus MUST be in the INT64 range. Sign is preserved and *c* MUST be a non-zero value.|
|OP_ADDDIV|220|0xdc|a b c|out|*a* is added to *b* then divided by *c* and the quotient of the division is returned. The intermediate sum CAN be in the INT128 range, while the quotient MUST be in the INT64 range. Sign is preserved and *c* MUST be a non-zero value.|
|OP_ADDMOD|221|0xdd|a b c|out|*a* is added to *b* then divided by *c* and the modulus of the division is returned. The intermediate sum CAN be in the INT128 range, while the modulus MUST be in the INT64 range. Sign is preserved and *c* MUST be a non-zero value.|

## Rationale

This section documents design decisions made in this specification.

### Complete set of composite operations for overflows

The set of four composite opcodes enables the full usecase, there is no need for `op_subdiv` and a corresponding `op_submod` as this can easily be done with
the composite addition opcodes just by fliping the sign of the second argument. This is illustrated in the `Math.cash` example library. 
The full set of composite opcodes is proposed instead of just the immediately useful `op_muldiv` for the more narrow AMM usecases to enable higher 
precision math which is useful much more broadly.

Trying to do int127 addition and subtraction without the composite addition opcodes is roundabout and complex, while technically possible without losing precision, it was found to be much cleaner to include it along side the composite multiplication opcodes to complete the set.

### Choice of codepoints

In the range of arithmetic opcodes there are 4 open slots but the possibility of re-enabling `OP_LSHIFT` & `OP_RSHIFT` makes this non-ideal.
The `OP_UNKNOWN` range presents a clean choice to put the four consecutive opcodes. The range `0xda` (218) through `0xdd` (221) was chosen to leave
a reasonable gap for possible future introspection opcodes, so they could be grouped together. 

## Evaluation of Alternatives

### Larger integers

An alternative solution for solving the overflow problem of 64-bit integer calculations is to simply extend the integer length.
However, even with larger integers there would still be a need for emulation of higher precision math.
This is illustrated by UniSwap V3 making good use using a `MulDiv` helperfunction despite the 256-bit integer precision of the EVM.
UniSwap can emulate `MulDiv` functionality with the native `mulmod` EVM opcode, it also makes use of the fact that the Ethereum math opcodes
such as multiplication do allow for overflow and return the higher-order bits.

If Bitcoin Cash upgrades to larger integers (e.g. 128-bit), then `op_muldiv` and `op_mulmod` should also use this same level of precision for the result
while allowing intermediate multiplication up to double the size (256-bit).

## Implementations

[TODO]

## Stakeholders & Statements

[TODO]

## Feedback & Reviews

- [`The need for additional Math opcodes`| bitcoincashresearch.org](https://bitcoincashresearch.org/t/the-need-for-additional-math-opcodes/1087/22)

## Changelog

- **v1.0.0 – 2022-7-07**
  - Expand CHIP to also include `op_adddiv` and `op_addmod`
  - change codepoints
  - include pseudo CashScript library for higher order math
- **v0.2.0 – 2022-7-06**
  - clarified higher precision addition/subtraction
  - rationale discussing adding the complete composite set
  - added easy overflow check to benefits
- **v0.1.0 – 2022-7-05**
  - first published draft version

## Copyright

This document is placed in the public domain.
