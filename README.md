- **TEP**: [0](https://github.com/ton-blockchain/TEPs/pull/0) *(don't change)*
- **title**: Extensible Huamn-Friendly Addresses
- **status**: Draft
- **type**: Meta
- **authors**: [Denis Subbotin](https://github.com/mr-tron), [Oleg Andreev](https://github.com/oleganza)
- **created**: 1.05.2023

# Summary

This proposal introduces new address format for all TON addresses.

# Motivation

There are several problems with current address format (aka "friendly format"):

* Misuse of a "bounceable" flag: apps and wallets converged on "non-bounceable" ("EQ...") format and wallets cannot trust it.
* Base64 encoding is not copy-paste friendly.
* Base64 encoding is not URL friendly and hence two flavors of the encoding are used creating confusion.
* Personal wallet addresses do not carry public key explicitly which hinders adoption of cryptographic features (e.g. encryption and ZK proofs).

New format addresses all these issues and is easy to deploy.

**Backward compatibility:** existing wallets that support TON DNS can resolve new addresses out of the box without any modification. This means, apps may publish their addresses in a new format right away.

**Forward compatibility:** new features could be gradually adopted within the proposed framework without breaking compatibility with existing software.

# Guide

Address format is a _TON DNS_ address string:

```
fybe9...6rg8e.wallet.ton
```

There are two important differences from regular DNS records:

1. There is no separate DNS record for the entire entry. There is entry for `wallet.ton` that resolves the entire address and returns required attributes directly without redirection to another record.
2. The address resolver `wallet.ton` is made immutable, which means the code for resolving the address can be cached indefinitely or even be made built-in, like you would expect from any address parser.

The address resolver takes care of parsing the address, verifying checksum and constructing necessary attributes: raw address for the contract, public key, bounceable flag, stateinit data etc.


# Specification

## Encoding

Address format generally consists of two parts: the *data part* and the *resolver*:

```
<data>.<resolver>.ton
```

It is permitted to have multiple levels of resolvers (e.g. `v4.wallet.ton` instead of `walletv4.ton`) or even multiple parts in the *data* part.

Resolvers are free to choose the encoding and the checksum algorithm. In this specification we propose default choice for the initial set of addresses:

## Suggested alphabet and checksum

Use Base 32 alphabet from [RFC 4648](https://datatracker.ietf.org/doc/html/rfc4648#section-7).
But all addresses SHOULD be converted to lower case. Upper case CAN be used for tecnical purpouse like "alphanum" mode in QR codes.

Standard addresses:
* Wallet address: workchain + pubkey (264 bits).wallet.ton
* Contract address: workchain + hash (264 bits).addr.ton

Checksum: pad with 16 bits to 280 bits (56 symbols in Base 32). Take first 16 bits from `cell_hash(cell(parsed_data || "resolver.ton" ))` where `"resolver.ton"` is a fully-qualified DNS name of the resolver contract.

For example:

`ovqqafgzellhvdvw6eyactxuuic5n3dwqyjtni6dgyxlnvgmntfqrgpp.addr.ton`



## Resolving an address

Fully compatible with [standard](https://github.com/ton-blockchain/TEPs/blob/master/text/0081-dns-standard.md)      


## Displaying addresses

Software that wishes to display an address MUST NOT ask contracts for constructing the address: that would be insecure — any contract would be able to pose as a legitimate address.

Software MUST use well-known address resolvers and attempt to construct addresses for each of them for a given contract, stopping at the first match.

To display an abbreviated address use leading 10 characters:

```
7kw508d6qejxqtdg4y5r3zadg4y5r3zarvary0c5xwejxt.wallet.ton

=>

7kw508d6qe
```

Abbreviated MUST NOT be used for any resolves - only displays because it's not cryptographically secure.

## DNS inegration.

### Bounce flag
Software MUST use record capability `cap_is_wallet#2177 = SmcCapability;` for setting `bounce` flag. 
Messages to record without this capability MUST have flag `bounce` = true. 
Messages to record without this capability MUST have flag `bounce` = false.

### Init state

//todo: write

# Drawbacks

Why should we *not* do this?

...

# Rationale and alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Prior art



# Unresolved questions

* Backresolve. How to display account address in explorers before it deployed?

# Future possibilities

