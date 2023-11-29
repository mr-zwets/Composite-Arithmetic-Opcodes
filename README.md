# CHIP-2023-07-Composite-Arithmetic-Opcodes

        Title: Composite Math Opcodes
        Type: Standards
        Layer: Consensus
        Owners: Mathieu Geukens & bitcoincashautist 
        Status: Draft
        Initial Publication Date: 2023-07-05
        Latest Revision Date: 2023-11-29

## Summary

This proposal adds four new math opcodes to the Bitcoin Cash Virtual Machine (BCH VM), `op_muldiv`, `op_mulmod`, `op_adddiv` and `op_addmod`.

`op_muldiv` greatly simplifies contracts using 64-bit integer multiplication in intermediate calculations.
These calculations currently fail on overflow so contract authors need to either restrict input ranges or use clever math workarounds which sacrifice precision.

Together, the set of composite opcodes allows contracts to emulate int127 math easily using 64-bit integers as demonstrated 
by the included `int127Math.cash` example CashScript Math library. This makes working with full precision calculations and 
working around overflows very easy with minimal overhead.

With retargetted VM limits it would become viable to emulate the functionality of these composite arthitmetic opcodes for 63-bit integers, as demonstrated in `emulatedOpcodes.cash`.

## Deployment

The proposal is still in draft phase and not ready for deployment.
## Motivation

Bitcoin Cash has upgraded to 64-bit integers and now with the advent of CashTokens the BCH VM enables complex DeFi applications,
this poses the need both for higher-order math and for allowing overflows in intermediate calculations.

Now, shortly after the CashTokens upgrade there are already multiple different Constant ProductMarket Maker (CPMM) contracts live on the network 
which carefully have to consider strategies for multiplication overflows.
The composite arithmetic opcode `op_muldiv` allows the intermediate calculation to overflow and hence simplifies the strategies dealing with overflows,
leading to simpler, more efficient and more capable smart contracts.

Moreover, the introduction of composite arithmetic opcodes for multiplication and addition enables convenient emulation of higher precision math in BCH contracts.
This way contract authors can work with full precision calculations and overflows easily and with minimal overhead.

## Benefits

By introducing these opcodes, the code for advanced contracts such as AMMs can be improved in multiple dimensions and 
more advanced and precise contracts using higher-order math emulation are enabled.

### Simpler, More Efficient & More Capable Contracts

There are two overflow strategies for contracts using mutiplication in an intermediate step: 
- Restrict the range of the coefficients so they can be guaranteed not to overflow 
- Use modulo math to approximate the end result of the calculation

In the former case, the introduction of `op_muldiv` would allow contract authors to expand the ranges of allowed coefficients easily.
In the latter  case, the introduction of `op_muldiv` would allow contract authors to simplify their contract by getting rid of the 
modulo math workaround and instead use the native functionality.

[Cauldron DEX](https://www.cauldron.quest/) is an example of the first strategy and [Fex.cash](https://fex.cash/) of the second one.
Both projects would benefit from having the option to utilize `op_muldiv` in a future version.
For Cauldron DEX this allows the contract to work with CashTokens with a very large supply (meaning either many decimals places or a very low value per token), whereas currently such pools would not work because of arithmetic overflows.
For Fex.cash this would mean a simpler (less error-prone) and more efficient contract both in terms of opcodes and bytesize.

UniSwap V3 also makes heavy use of a [`muldiv` helperfunction](https://docs.uniswap.org/contracts/v3/reference/core/libraries/FullMath) and has [taken initiative](https://ethereum-magicians.org/t/create-a-new-opcode-for-muldiv/8574) to get an `muldiv` opcode added to the EVM.

### Higher Precision Math

With `op_muldiv` and `op_mulmod` int127 multiplication can be easily emulated with full precision by using two 64-bit integers
to represent a int127 (as shown [here](https://medium.com/wicketh/mathemagic-full-multiply-27650fec525d)).
The addition and subtraction of two int127 numbers is made easy by the `op_adddiv` and `op_addmod` opcodes, 
however for division [CHIP-2021-05-loops: Bounded Looping Operations](https://github.com/bitjson/bch-loops) is required in addition to `op_muldiv` 
to make it practical (as shown [here](https://medium.com/wicketh/mathemagic-512-bit-division-in-solidity-afa55870a65)).

The four new composite arithmetic opcodes serve as primitives for building user-space libraries that enable higher order (int127) math.
CashScript is investigating [enabling support for libraries/macros](https://github.com/CashScript/cashscript/issues/153) 
so a user-space library for int127 math could be built and utilized by other contracts.
In this CHIP an example `Int127Math.cash` CashScript library is added to demonstrate the simplicity of int127 math with the introduction of the new opcodes.

### Easy overflow check

`op_muldiv` and `op_adddiv` serve as an easy overflow check for when a contract wants to check whether a multiplication, addition or subtraction would overflow so the contract can provide a separate codepath for this overflow case instead of instantly failing the transaction.
This overflow check is done by using by using the maximum integer as divisor in `op_muldiv` or `op_adddiv`, the result then returns a number `!=0` on overflow and `0` otherwise. See the following CashScript code as an example:

```
  bool mulOverflows = muldiv(a, b, maxint) != 0;
  if(mulOverflows){
    ...
  } else {
    ...
  }
```

If contracts currently want to implement such an overflow check they need to emulate higher order math, which can be worked out as described in `emulatedOpcodes.cash`.

## Costs & Risk Mitigation

The following costs and risks have been assessed.

### Modification to Transaction Validation

The addition of new opcodes has the potential to increase worst-case transaction validation costs and expose VM implementations to Denial of Service (DOS) attacks.

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

The set of four composite opcodes enables int127 math in a very clean and concise way, as shown by the `Int127Math.cash` example library. There is no need for a separate `op_subdiv` and a corresponding `op_submod` as this can easily be done with
the composite addition opcodes by just fliping the sign of the second argument.
This CHIP proposes to add this complete set of composite opcodes instead of just the immediately useful `op_muldiv` for the more narrow CPMM usecases to enable higher precision (int127) math which is useful much more broadly.

### Choice of codepoints

In the range of arithmetic opcodes there are 4 open slots but the possibility of re-enabling `OP_LSHIFT` & `OP_RSHIFT` makes this non-ideal.
The `OP_UNKNOWN` range presents a clean choice to put the four consecutive opcodes. The range `0xda` (218) through `0xdd` (221) was chosen to leave
a reasonable gap for possible future introspection opcodes, so they could be grouped together. 

## Evaluation of Alternatives

### Emulating opcode functionality

With a proposal such as [CHIP 2021-05 Targeted Virtual Machine Limits](https://bitcoincashresearch.org/t/chip-2021-05-targeted-virtual-machine-limits/437), it becomes practically feasable to emulate `muldiv`,`mulmod`,`adddiv` & `addmod` functionality.
This is demonstrated in the `emulatedOpcodes.cash` example CashScript library, which only uses the current BCH VM capabilities.
This emulation is not possible inside a single smart contract with the current VM limits (201 opcode & 520 bytesize limits) but by reworking the VM limits, similar functionality as proposed in the CHIP can be achieved with some important caveats.

A first limitation is that because the emulation works by splitting each integer into two 32-bit parts it can only represent up-to 63-bit integers to use with the emulated `muldiv`,`mulmod`,`adddiv` & `addmod` which in turn can be use to emulate 125-bit integer math (instead of the 127-bit math by having the native opcodes). A second limitation is that contract developers need to provide the result of the emulated arithmetic already, the contract just verifies this calculation, this is because it is not possible to emulate higher order division without loops and bitshifts. 

Emulating the composite arithmetic opcodes would increase the size of smart contracts, especially when contracts make repeated use of them, for example by emulating higher order math. Another important consideration is the increased complexity that comes along with emulation but with good library support in Cashscript much of the complexity can be hidden from developers using Math helper libraries.

### Larger integers

An alternative solution for solving the overflow problem of 64-bit integer calculations is to simply extend the integer length.
However, even with larger integers there would still be a need for emulation of higher precision math.
As shown by UniSwap V3 still using a `MulDiv` helperfunction despite the 256-bit integer precision of the EVM.

If Bitcoin Cash upgrades to larger integers (e.g. 128-bit), then `op_muldiv` and `op_mulmod` should also use this same level of precision for the result
while allowing intermediate multiplication up to double the size (256-bit).

### ADDDIVMOD and MULDIVMOD

A proposed alternative is having `ADDDIVMOD`and `MULDIVMOD` which just combines the `adddiv`, `addmod` and the `muldiv`, `mulmod` 
functionality respectively. The reasoning is that only `muldiv` would see standalone usage whereas `adddiv`, `addmod` and `mulmod` 
would only be used in this combined way.
It requires estimation of how much int127 math would be used compared to standalone `muldiv` usage to say what the better alternative is.

## Implementations

[TODO]

## Stakeholders & Statements

> I’m in support of this CHIP. The size of a liquidity pool in the Cauldron DEX contract is currently limited by the size of int64.
>
> This can be worked around today by reducing precision (higher spread between buy and sell). With OP_MULDIV, the precision loss would be significantly less.

— Dagurval, Bitcoin Cash developer and creator of Cauldron DEX

## Feedback & Reviews

- [`The need for additional Math opcodes`| bitcoincashresearch.org](https://bitcoincashresearch.org/t/the-need-for-additional-math-opcodes/1087/22)

## Changelog

- **v1.2.1 – 2022-11-29**
  - improve section on alternatives 
  - added to statements
  - repeat draft status in deployment section
- **v1.2.0 – 2022-11-24**
  - fixes to emulated opcodes library
  - clarify that emulated opcodes can only do int63, and emulate int125 math
- **v1.1.1 – 2022-7-28**
  - clarified easy overflow check section
- **v1.1.0 – 2022-7-27**
  - added Higher script limits to evaluated alternative
  - added `emulatedOpcodes.cash` example CashScript library
  - changed the `Math.cash` file to `Int127Math.cash`
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
