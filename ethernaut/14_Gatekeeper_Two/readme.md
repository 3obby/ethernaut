***Gatekeeper Two***

This gatekeeper introduces a few new challenges. Register as an entrant to pass this level.

Things that might help:

- Remember what you've learned from getting past the first gatekeeper - the first gate is the same.

- The assembly keyword in the second gate allows a contract to access functionality that is not native to vanilla Solidity. See here for more information. The extcodesize call in this gate will get the size of a contract's code at a given address - you can learn more about how and when this is set in section 7 of the yellow paper.

- The ^ character in the third gate is a bitwise operation (XOR), and is used here to apply another common bitwise operation (see here). The Coin Flip level is also a good place to start when approaching this challenge.

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract GatekeeperTwo {

  address public entrant;

  modifier gateOne() {
    require(msg.sender != tx.origin);
    _;
  }

  modifier gateTwo() {
    uint x;
    assembly { x := extcodesize(caller()) }
    require(x == 0);
    _;
  }

  modifier gateThree(bytes8 _gateKey) {
    require(uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == type(uint64).max);
    _;
  }

  function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
    entrant = tx.origin;
    return true;
  }
}
```

**`gateOne()` is trivial. We can simply deploy a proxy contract.**

`gateTwo()` has some new properties, let's take a look:

```
uint x;
assembly { x := extcodesize(caller()) }
require(x == 0);
```
`assembly{}`: Inline assembly is a way to access the Ethereum Virtual Machine at a low level. This bypasses several important safety features and checks of Solidity. You should only use it for tasks that need it, and only if you are confident with using it.

An inline assembly block is marked by `assembly { ... }`, where the code inside the curly braces is code in the ***Yul*** language.

`caller()`: call sender (excluding `DELEGATECALL`)

`:=`: assignment

`extcodesize(a)`: size of the code at address a

What does `caller()` return in Yul when `DELEGATECALL` is used? It returns an empty number (0x000..0000).

All that being said, we now know that **we can use a `DELEGATECALL` to force `caller()` to return a `extcodesize` of 0, fulfilling the requirements for `gateTwo()`.**

`gateThree()` requires us to construct a key of format bytes8 that has the following requirement:

```
uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == type(uint64).max
```

Sheesh. Let's start with the right side since that's smaller.

`type(T).max`: The largest value representable by type T. This is to say, in binary form, 64 bits maxxes out at `1111111111111111111111111111111111111111111111111111111111111111`. In base 10, this is 18446744073709551615. In Hexadecimal, this is FFFFFFFFFFFFFFFF.

On the left side, basically we need to calculate a `_gateKey` that matches that number. The `^` isn't an exponent, it's the bitwise operation (XOR).

In simple terms, if we line up two binaries, the result of each position is 1 if ***only one*** of the inputs is 1. Here's an example:

```
a = 5;        // 0101
b = 3;        // 0011

(a ^ b);      // 0110
// output: 6
```

Now that we know this, we know that we'll need to calculate a key that compliments whatever the result of `uint64(bytes8(keccak256(abi.encodePacked(msg.sender))))` is. For example:

```
result:    00101011
key:       11010100

combined:  11111111
```
