- **TEP**: [0](https://github.com/ton-blockchain/TEPs/pull/0) *(don't change)*
- **title**: Extensible Human-Friendly Addresses
- **status**: Draft
- **type**: Meta
- **authors**: [Denis Subbotin](https://github.com/mr-tron), [Oleg Andreev](https://github.com/oleganza)
- **created**: 1.05.2023

# Summary

This proposal introduces new address format for all TON addresses.

# Motivation

There are several issues with current address format (aka "friendly format"):

* Misuse of a "bounceable" flag: apps and wallets historically converged on a "non-bounceable" ("EQ...") format that made the flag useless.
* Base 64 encoding is not copy-paste friendly: it is common to have trailing underscores or dashes that can be confused with ordinary punctuation.
* Base 64 encoding is not URL-friendly: two flavors of the encoding are used adding to confusion.
* Personal wallet addresses do not carry public key explicitly which hinders adoption of cryptographic features (e.g. encryption and ZK proofs).

The goal of this standard is to address all these issues and make it easy to deploy.

**Backward compatibility:** existing wallets that support TON DNS can resolve new addresses out of the box without any modification. This means, apps may publish their addresses in a new format right away.

**Forward compatibility:** new features could be gradually adopted within the proposed framework without breaking compatibility with existing software.

# Guide

Address format is a _TON DNS_ address string that looks like this:

```
f1ex9...6rg8e.wallet.ton
```

There are two important differences from regular DNS records:

1. There is no individual DNS record for the entire entry. There is a _resolver_ entry for `wallet.ton` that resolves the entire address and returns required attributes directly without redirection to another record.
2. The address resolver `wallet.ton` is designed to be immutable, which means the code for resolving the address can be cached indefinitely or even be made built-in, like you would expect from any address parser.

The address resolver takes care of parsing the address, verifying the checksum and constructing the necessary attributes: raw address for the contract, public key, bounceable flag, stateinit data etc.


# Specification

## Encoding

Address format generally consists of two parts: the *data part* and the *resolver*:

```
<data>.<resolver>.ton
```

It is permitted to have multiple levels of resolvers (e.g. `v4.wallet.ton` instead of `walletv4.ton`) or even multiple parts in the *data* part.

Data part MUST use Base 32 alphabet from [RFC 4648, section 6](https://datatracker.ietf.org/doc/html/rfc4648#section-6).

Implementations MUST accept the addresses in both lower and upper case.

Implementations MUST convert addresses to lower-case before resolving or displaying them to the users. This is a requirement of the [TON DNS standard](https://github.com/ton-blockchain/TEPs/blob/master/text/0081-dns-standard.md).

Upper case CAN be used in specific contexts such as "alphanum" mode in QR codes.

## Checksum

Resolvers are free to choose the checksum algorithm and the size of the checksum. 
The size may be adjusted to ensure 40-bit alignment (this is because Base 32 alphabet uses 5-bit symbols, while TON DNS is byte-aligned) of the resulting string.

## Resolving an address

Resolution follows the [TON DNS standard](https://github.com/ton-blockchain/TEPs/blob/master/text/0081-dns-standard.md), with additional checks on 

## Standard Addresses

This specification proposes two kinds of addresses: specific addresses for standard wallets and generic addresses for any contracts.

Standard addresses have 280-bit data part, with 8 bits for a workchain ID, 256 bits for a contract hash or a pubkey, and 16 bits for a checksum.

* Wallet address: (workchain + pubkey + checksum).walletNM.ton (`wallet31` for "v3R1" contract, `wallet42` for "v4R2" contract etc.)
* Contract address: (workchain + hash + checksum).addr.ton

Checksum is computed on the reverse "internal representation": `ton\0addr\0...\0`.

1. Split the reverse DNS name in three parts: `x` prefix, then 16-bit checksum `c` and trailing zero 8 bits `\0`.
2. Let `h` be the `sha256(x)`.
3. Take the first 16 bits of `h` and compare it with `c`.
4. If the bits match, the checksum is correct.

Examples:

`ovqqafgzellhvdvw6eyactxuuic5n3dwqyjtni6dgyxlnvgmntfqrgpp.addr.ton`


## Displaying addresses

Implementations MUST NOT ask contracts for constructing the address: that would be insecure — any contract would be able to pose as a legitimate address.

Implementations MUST use well-known address resolvers and attempt to construct addresses for each of them for a given contract, stopping at the first match.

Implementations MUST convert the address to lower-case before displaying it.

Implementations MAY use the leading 10 characters when displaying an abbreviated address:

```
7kw508d6qejxqtdg4y5r3zadg4y5r3zarvary0c5xwejxt.wallet.ton

=>

7kw508d6qe
```

Abbreviated form MUST NOT be used for resolving an address.


## DNS integration

### Bounceable Flag

Software MUST use record capability `cap_is_wallet#2177 = SmcCapability;` for setting the `bounceable` flag. 

If this capability is present, implementations MUST set the `bounceable` flag to `false`. Otherwise, it MUST be set to `true`. 

### Init state

Add new capability `cap_state_init#4571 init:StateInit = SmcCapability;` for returning state init which MUST be attached to outgoing message. 

#### Rationale 

- It reduces number of accounts without uninitialized accounts which partialy remove problem with backresolve.
- It allows to create addresses like `1000000000-1684273816-1234.music.invoices.ton` for creating one-time-payment invoice to Music Shop for 1 TON with ttl 10 minutes and uniq id 1234.

# Drawbacks

Why should we *not* do this?

TBD.

# Rationale and alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Prior art

TBD.

# Unresolved questions

* Backresolve. How to display account address in explorers before it deployed?

TBD.

# Future possibilities

TBD.
