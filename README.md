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
wA1let...6rg8e.addr.ton
```

There are two important differences from regular DNS records:

1. There is no individual DNS record for the entire entry. There is a _resolver_ entry for `addr.ton` that resolves the entire address and returns required attributes directly without redirection to another record.
2. The address resolver `addr.ton` is designed to be immutable, which means the code for resolving the address can be cached indefinitely or even be made built-in, like you would expect from any address parser.

The address resolver takes care of parsing the address, verifying the checksum and constructing the necessary attributes: raw address for the contract, public key, bounceable flag, stateinit data etc.


# Specification

## Encoding

Address format generally consists of two parts: the *data part* and the *resolver*:

```
<data>.<resolver>.ton
```

It is permitted to have multiple levels of resolvers (e.g. `v4.addr.ton` instead of `addrv4.ton`) or even multiple parts in the *data* part.

Data part MUST use Base 32 alphabet from [RFC 4648, section 6](https://datatracker.ietf.org/doc/html/rfc4648#section-6).

Implementations MUST accept the addresses in both lower and upper case.

Implementations MUST convert addresses to lower-case before resolving or displaying them to the users. This is a requirement of the [TON DNS standard](https://github.com/ton-blockchain/TEPs/blob/master/text/0081-dns-standard.md).

Upper case CAN be used in specific contexts such as "alphanum" mode in QR codes.

## Checksum

Resolvers are free to choose the checksum algorithm and the size of the checksum.
The size may be adjusted to ensure 40-bit alignment (this is because Base 32 alphabet uses 5-bit symbols, while TON DNS is byte-aligned) of the resulting string.

## Resolving an address

From the perspective of the user, nme resolution simply follows the [TON DNS standard](https://github.com/ton-blockchain/TEPs/blob/master/text/0081-dns-standard.md). 
The contract `addr.ton` returns attributes that include the full address and potentially, a public key.

Inside the `addr.ton` contract the following protocol is applied to the subdomain _name_ in the `dnsresolve` get-method:

1. Convert the name string from base32 to a binary string. Use `0` as a padding symbol. (Note: we may introduce some ambiguity to the address encoding if we don't force those to be in the end.)
2. Read the first 8 bits of the binary name stored in slice `s` as a 256-bit unsigned integer `d` (`d <= 255`).
3. Load a _parser dictionary_ from the storage: `M`
4. Locate the cell in the dictionary `M` by key `d`.
5. Cast the cell into continuation and execute it with the following stack order: `M d s`.
6. After execution, the stack should contain two values `8*m` and `Cell`: resolved bitlength and TON.DNS record or dictionary cell per [dnsresolve standard](https://github.com/ton-blockchain/TEPs/blob/master/text/0081-dns-standard.md).
7. Return `8*m` and `Cell`.

Note that addr.ton does not make assumption about the exact length of the name. Also, additional records may be stored in the dictionary `M` beyond 8-bit range for further extensibility.


## Standard Addresses

This specification proposes a generic mechanism that permits resolution of a number of standard addresses for apps and wallets.

Standard addresses have 280-bit data part: 8 bits for address discriminant, 256 bits for the data part (contract hash or a pubkey), and 16 bits for a checksum.

```
discriminant (u8) ++ data (u256) ++ checksum (u16) ++ `addr.ton`
```

Standard discriminants:

Base32 prefix | Binary prefix | Hex prefix    | Workchain | Contract type 
--------------|---------------|---------------|-----------|-----------------
w             |  1011 0000    | 0xb1          | 0         | wallet v3r2     
w             |  1011 0001    | 0xb2          | 0         | wallet v4r2
w             |  1011 0010    | 0xb3          | 0         | wallet v5r1
g             |  0011 0000    | 0x31          | -1        | wallet v3r2     
g             |  0011 0001    | 0x32          | -1        | wallet v4r2
g             |  0011 0010    | 0x33          | -1        | wallet v5r1
d             |  0001 1000    | 0x18          | 0         | other contracts
t             |  1001 1000    | 0x98          | -1        | other contracts

The letters are chosen according to the mnemonic rules:

* `w` — wallet
* `g` — global (wallet on masterchain)
* `d` — dapp
* `t` — TON

Checksum is computed as follows:

1. Read the discriminant, data and checksum in distinct integers.
2. Write the discriminant and data in a 264-bit slice `x`.
3. Let `h` be the `sha256(x)`.
4. Take the first 16 bits of `h` and compare it with `c`.
5. If the bits match, the checksum is correct.

Example:

`wnrtwmvo5dcy47ov76wi76yofxhkc44bxhpk3pxpzl7lvpw6vw7o6btg.addr.ton`

Notes:

1. The proposed discriminant allows for up to 5 more upgrades to the wallet retaining prefix `w`/`g`.
2. A third workchain can be supported by adding another discriminant for the relevant wallet formats. Such workchain may not be even TVM-compatible, so existing wallet versions may not be relevant to it. Generic contract address may also require differently-sized encoding and therefore handled by another discriminant.
3. When running out of 256 discriminants, another parser could be added that uses an extended discriminant.


## Displaying addresses

Implementations MUST NOT ask contracts for constructing the address: that would be insecure — any contract would be able to pose as a legitimate address.

Implementations MUST use well-known address resolvers and attempt to construct addresses for each of them for a given contract, stopping at the first match.

Implementations MUST convert the address to lower-case before displaying it.

Implementations MAY use the leading 10 characters when displaying an abbreviated address:

```
wnrtwmvo5dcy47ov76wi76yofxhkc44bxhpk3pxpzl7lvpw6vw7o6btg.addr.ton

=>

wnrtwmvo5d
```

Abbreviated form MUST NOT be used for resolving an address.


## Extending addr.ton

Addr.ton logic is designed to be immutable for security, but also extensible. The extension logic is designed to avoid any impact on pre-existing addresses and only be able to add new formats. The worst-case scenario is the loss of ability to upgrade, in which case another TON.DNS TLD can be used as an alternative.

Addr.ton supports two storage-modifying tools:

* Add parser: adds a parser code to a slot with unused number. If the slot is allocated already, fails.
* Change controller: changes the address of the controlling contract. This allows key rotation or switching from a plain wallet to a DAO.

To ensure stability, the contract allows assigning a value to the dictionary only once. Then it stays immutably in the dictionary forever. 

To prevent potential abuse of the storage, the upgrade method only allows 16-bit keys for the parsers: that is allowing additional 8 bits for expansion if the 8-bit discriminant will be deemed insufficient.

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
