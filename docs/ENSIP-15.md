# ENSIP-15: Off-chain ENS Records Manager

### Resolver with Gasless ENS Records Management using IPNS and CCIP-Read

| &nbsp; | &nbsp; |
| -- | -- |
| **Author**    | [freetib.eth](@0xc0de4c0ffee), [sshmatrix.eth](@sshmatrix), [eth.limo](@eth_limo) |
| **Status**    | Draft |
| **Submitted** | 2023-04-10 |

## Abstract

This ENSIP introduces an optional specification for Resolvers to fetch and render ENS records with `CCIP-Read` (ENSIP-10) using IPNS and RFC-8615 `.well-known` standard. The outlined implementation doesn't need web2 gateways to relay off-chain records and relies on users to host their records themselves using IPNS, resulting in a gasless and autonomous implementation. This specification is fully optional and ENS users must manually switch their Resolver address to enable the features included therein.

## Motivation

ENS has the potential to revolutionise decentralised access to the web with an attached identity system through linked records, but the usage of ENS records seems to have pleatued. This is a consequence of intrinsically high gas costs associated with adding and updating the records since there are no additional fees to set records. While IPNS has reduced the gas costs associated with updating `contenthash`, other records appear to either be unset or rarely updated once set.

This specification solves the aforementioned problem of high gas costs by storing the ENS records off-chain inside `.well-known` (RFC8615) directory of the users' IPNS contenthash. Records stored under `.well-known` standard can then be queried through [ENSIP-10 `CCIP-Read`](https://docs.ens.domains/ens-improvement-proposals/ensip-10-wildcard-resolution) implemented in this specification. The implementation outlined here doesn't require any additional gateways to fetch and render the ENS records, and the user is fully in control of their records, e.g. hosting the records on IPFS and linking the IPFS hash to their IPNS key. With this method, users are able to update their records as often as possible at no cost whatsoever without ever losing custody of their data (see figure below). We believe that this specification will enable frequent updates of records other than the `contenthash` and propel ENS adoption as an identity layer.

![](https://raw.githubusercontent.com/namesys-eth/ccip2-resources/main/graphics/png/ccip2.png)

## Specification

The proposed Resolver and ENS records storage outlined in this document expects the following implementation:

### a) CCIP-Read Resolver (ENSIP-10)

This specification is an extension of ENSIP-10 `CCIP-Read`

```solidity
function resolve(bytes calldata name, bytes calldata data) external view returns(bytes memory)
```

where, the `_path` to query an ENS record and the full `_domain` as string shall be decoded from encoded `name` variable using `DNSDecode()` function:

```solidity
function DNSDecode(
    bytes calldata name
) public view returns (
    string memory _domain, string memory _path, string memory _label
){
    uint level = 1; // domain heirarchy level
    uint i = 1; // counter
    uint len uint8(bytes1(name[:1])); // length of label
    _label = string(name[1: i += len]); // final value is TLD ".eth"
    _path = _label; // suffix after /.well-known/
    _domain = _label; // full domain as string

    while(name[i] > 0x0) { // DNS Decode
        len = uint8(bytes1(name[i: ++i]));
        _label = string(name[i: i += len]);
        _domain = string.concat(_domain, ".", _label);
        _path = string.concat(_label, "/", _path);
        ++level;
    }
}
```

### b) Off-chain Records Storage

For this specification to make pratical sense, we expect the `contenhash` to be of IPNS type. IPNS hashes are key-based decentralised storage pointers that only need to be added once to on-chain storage by the user. IPNS hashes can in turn serve as proxy and point to upgradeable IPFS or IPLD content. In the parent IPNS directory, the records must be stored in the [RFC-8615](https://www.rfc-editor.org/rfc/rfc8615) compliant `.well-known` directory format. ENS records for any name `sub.domain.eth` must then be stored in JSON format under a [reverse-DNS](https://en.wikipedia.org/wiki/Reverse_domain_name_notation) type directory path using `/` instead of `.` as separator, i.e. in format `.well-known/eth/domain/sub/<record>.json`.

**1. Some Examples:**

- ENS text record for `vitalik.eth`'s avatar is stored at `https://vitalik.eth.limo/.well-known/eth/vitalik/avatar.json` formatted as

```solidity
{ data: abi.encode(string("eip155:1/erc1155:0xb32979486938aa9694bfc898f35dbed459f44424/10063")) }
```

- ETH address record for `sub.domain.eth` is stored at `https://sub.domain.eth/.well-known/eth/domain/sub/addr-60.json` formatted as

```solidity
{ data: abi.encode(<addr-60>) }
```

Note: If the JSON data is signed by the Registrant of `domain.eth`, it must be prefixed with `bytes4` of `callback` function selector as,

```solidity
{ data: bytes.concat(Resolver.___callback.selector, <signed_data>}
```

**2. Resolver function → JSON file names:**

| Type | Function | JSON file |
| -- | -- | --- |
| Text Records ([ENSIP-05](https://docs.ens.domains/ens-improvement-proposals/ensip-5-text-records)) | `text(bytes32 node, string memory key)` | `<key>.json` |
| Ethereum Address | `addr(bytes32 node)` | `addr-60.json` |
| Multichain Address ([ENSIP-09](https://docs.ens.domains/ens-improvement-proposals/ensip-9-multichain-address-resolution)) | `addr(bytes32 node, uint coinType)`| `addr-<coinType>.json` |
| Public Key | `pubkey(bytes32 node)`| `pubkey.json` |
| Contenthash ([ENSIP-07](https://docs.ens.domains/ens-improvement-proposals/ensip-7-contenthash-field)) | `contenthash(bytes32 node)` | `contenthash.json` |


### CCIP Gateways

| Type | Identifier | Gateway URL |
| --- | --- | --- |
| `ipns://<contenthash>` | `0xe5` | `https://<base36-CID-v1>.ipns.dweb.link/.well-known/..` |
| `ipfs://<contenthash>` | `0xe3` | `https://<base32-CID-v1>.ipfs.dweb.link/.well-known/..` |
| ENS + IPNS Node| &nbsp; | `https://domain-eth.ipns.dweb.link/.well-known/..` |
| ENS | &nbsp; | `https://domain.eth.limo/.well-known/..` |
| ENS + IPFS2 resolver| `0xe3`, `0xe5` | `https://<CID-v1>.ipfs2.eth.limo/.well-known/..` |

## Code

### --

```solidity
	//...

	funMap[iResolver.addr.selector] = "addr-60"; // eth address
	funMap[iResolver.pubkey.selector] = "pubkey";
	funMap[iResolver.name.selector] = "name";

	//...

	bytes4 fun = bytes4(data[: 4]); // 4 bytes identifier

	if (fun == iResolver.contenthash.selector) {
		if (level == 3) resolveContenthash(labels[0]);
		__lookup(HomeContenthash);
	}

	string memory jsonFile;
	if (fun == iResolver.text.selector) {
		jsonFile = abi.decode(data[36: ], (string));
	} else if (fun == iOverloadResolver.addr.selector) {
		jsonFile = string.concat(
			"addr-",
			uintToNumString(abi.decode(data[36: ], (uint)))
		);
	} else {
		jsonFile = funMap[fun];
		require(bytes(jsonFile).length != 0, "Invalid Resolver Function");
	}
```

### --

```solidity
	function resolve(bytes calldata name, bytes calldata data) external view returns(bytes memory) {
        uint level;
        uint len;
        bytes[] memory labels = new bytes[](3);
        //string memory _path;
        // dns decode
        for (uint i; name[i] > 0x0;) {
            len = uint8(bytes1(name[i: ++i]));
            labels[level] = name[i: i += len];
            //_path = string.concat(string(labels[level]), "/", _path);
            ++level;
        }
        bytes4 fun = bytes4(data[: 4]); // 4 bytes identifier
        if (fun == iResolver.contenthash.selector) {
            if (level == 3)
                resolveContenthash(labels[0]);

            __lookup(HomeContenthash);
        }
        string memory jsonFile;
        if (fun == iResolver.text.selector) {
            jsonFile = abi.decode(data[36: ], (string));
        } else if (fun == iOverloadResolver.addr.selector) {
            jsonFile = string.concat(
                "addr-",
                uintToNumString(abi.decode(data[36: ], (uint)))
            );
        } else {
            jsonFile = funMap[fun];
            require(bytes(jsonFile).length != 0, "Invalid Resolver Function");
        }

        string memory _prefix;
        if (level == 3) {
            _prefix = string.concat(
                "https://",
                string(labels[0]),
                ".",
                string(labels[1]),
                ".eth"
            );
        } else {
            _prefix = string.concat("https://", string(labels[0]), ".eth");
        }
        revert OffchainLookup(
            address(this), // callback contract
            listGate(_prefix, jsonFile), // gateway URL array
            "", // {data} field, blank//recheck
            IPFS2.__callback.selector, // callback function
            abi.encode( // extradata
                block.number, // checkpoint
                keccak256(data), // namehash + calldata
                keccak256(
                    abi.encodePacked(
                        blockhash(block.number - 1),
                        address(this),
                        msg.sender,
                        keccak256(data)
                    )
                )
            )
        );
    }

```
