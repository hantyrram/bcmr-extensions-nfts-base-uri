# CHIP-2026-11-NftsBaseUri: NFTs Base URI Extension for BCMR


* **Title:** NFT Types Base URI Extension for BCMR
* **Type:** Standards
* **Layer:** Applications
* **Status:** Draft
* **Initial Publication Date:** 2026-11-01
* **Owner:** Ronaldo Ramano
* **Created:** 2026-11-01

## Summary

This proposal introduces a standardized extension for the Bitcoin Cash Metadata Registry (BCMR) called `nftsBaseUri`. This extension allows issuers to completely decouple Nft metadata identifiers from their metadata values by routing resolution to an external deterministic URI scheme (`baseUri`). This minimizes BCMR file sizes and mirrors efficient metadata resolution frameworks found in wider crypto ecosystems while remaining 100% backward compatible with legacy BCMR parsers.

## Motivation

Under the current BCMR specification, explicitly defining metadata for individual NFTs requires inserting an entry for every unique commitment inside the `types` object. For collections containing thousands of unique items, this architectural requirement causes severe performance bottlenecks:

1. **Registry Bloat:** BCMR JSON files scale linearly with collection size, easily reaching tens of megabytes, which exhausts resources on mobile wallets and client-side indexers.
2. **Redundant Infrastructure:** Issuers are forced to duplicate information inside the BCMR file that already exists on decentralized storage networks (like IPFS) or remote APIs.
3. **Immutability Friction:** Updating global mechanics or attributes requires re-signing and re-propagating a massive registry file rather than modifying localized data definitions.

By introducing an extension that delegates resolution via string templates, registries remain lightweight, scalable, and highly performant.

## Technical Specification

A new schema object namespace key MUST be recognized inside the BCMR identity-level `extensions` block: `nftsBaseUri`.

### Schema Definition

The `nftsBaseUri` object contains the following property:

| Property | Type | Presence | Description |
| :--- | :--- | :--- | :--- |
| `baseUri` | `string` | **Required** | The base protocol location string (e.g., `ipfs://`, `ipns://`, `https://`) pointing to either an external routing directory or a consolidated master JSON file. |

# Definition: `NftTypeKey`

The **`NftTypeKey`** is the standardized string identifier used to query and locate a unique metadata profile. Depending on type of Nft collection i.e. SequentialNftCollection or ParsableNftCollection, it evaluates to either the raw token commitment or an execution-computed bytecode parameter. 

* **SequentialNftCollection:** The `NftTypeKey` is the raw token commitment.
* **ParsableNftCollection:** The `NftTypeKey` is the execution-computed bytecode parameter i.e. the bottomAltStackHex value.

### Architectural Structure Example

```json
{
  ...,
    "identities": {
      "0000000000000000000000000000000000000000000000000000000000000000": {
        "2026-11-01T00:00:00Z": {
          "name": "Ecosystem Dynamic Types Collection",
          "symbol": "EDTC",
          "extensions": {
            "nftsBaseUri": {
              "baseUri": "ipfs://QmXoypizjW3WknFiJnKLwHCnL72vedxjQkDDP1mXWo6uco/"
            }
          }
        }
    }
  }
}
```

## Resolution Algorithm

When a compatible wallet or indexer encounters a CashToken NFT output belonging to a category that declares an `nftsBaseUri` extension, it MUST evaluate the `baseUri` topology and determine the metadata using the following sequence:

### 1. Extract and Normalize Identifier Key
* Retrieve the NFT commitment. 
* Check the NftCategory.parse.bytecode to determine if it's a SequentialNftCollection or ParsableNftCollection 
* If it's a SequentialNftCollection, use the NFT commitment as the `NftTypeKey` 
* If it's a ParsableNftCollection, evaluate the NFT commitment using the NftCategory.parse.bytecode and use the bottomAltStackHex as the `NftTypeKey`

### 2. Determine Topology and Parse
All resolved endpoints MUST return valid JSON payloads. Regardless of whether directory routing or consolidated mapping is used, the final payload extracted for a given `NftTypeKey` **MUST resolve to a schema-valid `NftType` object** as defined in the official core BCMR specification (containing fields such as `name`, `description`, `uris`, etc.).

* **Scenario A: Directory Router Routing (Trailing Slash `/`)**  
  If the `baseUri` terminates with a `/`, the client constructs a distinct network request targeting that asset endpoint directly:  
  `Target_URL = baseUri + NftTypeKey`  
  *The JSON file returned by this endpoint MUST be a valid `NftType` object.*

* **Scenario B: Centralized Consolidated File Routing (No Trailing Slash)**  
  If the `baseUri` does not terminate with a `/`, the client MUST treat the URI as a path to a single centralized JSON dictionary file containing all type pairs.  
  1. Fetch the master file at `baseUri`.
  2. Parse the fetched object and locate the properties nested under the `NftTypeKey` string key locally.  
  *The object nested under the `NftTypeKey` key MUST be a valid `NftType` object.*

### Resolution Examples

| `baseUri` | Path Context | Extracted Identifier Hex | Final Resolution Behavior |
| :--- | :--- | :--- | :--- |
| `ipfs://QmDirectoryHash/` | Directory (Static) | `01a3ff` | HTTP GET to `ipfs://QmDirectoryHash/01a3ff` (Returns an `NftType` schema) |
| `ipns://k51qzi.../` | Directory (Dynamic) | `01a3ff` | Resolve IPNS pointer to IPFS directory CID, then fetch `/01a3ff` (Returns an `NftType` schema) |
| `https://site.com` | API Endpoint | `b45f` | HTTP GET to `https://site.comb45f` (Returns an `NftType` schema) |
| `ipfs://QmConsolidatedFileHash` | Single Central File | `01a3ff` | Fetch `QmConsolidatedFileHash`, lookup key `"01a3ff"` (Yields an `NftType` schema) |

## Managing Updates and Additions in Decentralized Environments

Because raw IPFS directory hashes (`ipfs://Qm...`) are strictly immutable, adding a new JSON metadata file into an active repository shifts the resulting root hash of that directory. Issuers looking to mint additional token types incrementally without updating the BCMR file itself MUST choose one of the following directory upgrade path workflows:

### A. The InterPlanetary Name System Pointer (IPNS) Protocol
Issuers may register an `ipns://` address within the `baseUri` key. 
1. The issuer generates a persistent cryptographic keypair that acts as a fixed pointer name (e.g., `ipns://k51qz.../`).
2. When new NFT variations are minted, the issuer uploads the individual file metadata, links it with the older directory tree graph via object patching, and publishes the *new* resulting root folder hash onto that unchanging IPNS index.
3. Wallets automatically follow the IPNS record to the updated target, tracking additions transparently over time.

### B. Dynamic Object Patch Routing
If an issuer decides to use a raw immutable `ipfs://` root block, adding a new type file requires rebuilding a link table manually or via CLI tooling:
```bash
ipfs object patch add-link <EXISTING_DIRECTORY_CID> <NEW_COMMITMENT_HEX> <NEW_FILE_METADATA_CID>
```
Executing this operation returns a completely new directory hash structure containing all preceding entries plus the newly targeted type item. The issuer then records this updated CID string in an updated, timestamped `IdentitySnapshot` entry inside the BCMR repository graph.

## Centralized File Schema (Scenario B Output Standard)
When utilizing a single centralized file router strategy, the hosted file structure at `baseUri` MUST conform to a top-level mapping arrangement where each key holds an explicit `NftType` payload:

```json
{
  "00": {
    "name": "Default Category Fallback Token Asset",
    "description": "Base attributes go here"
  },
  "01a3ff": {
    "name": "Warrior Item Variant #1",
    "description": "Decoupled specialized metadata payload",
    "uris": {
      "icon": "ipfs://QmIconHash"
    }
  }
}
```

## Backward Compatibility & Fallback

This extension enforces strict backward compatibility:
* **Legacy Wallets:** Will ignore the `extensions.nftsBaseUri` block entirely. They will fall back to using the base identity-level metadata (Name and Symbol) to render the token, preventing client crashes or registry rejection.
* **Coexistence Policy:** Issuers may choose to populate both the `nftsBaseUri` block AND standard minimal `types` entries to offer an explicit basic name fallback for older wallets while serving deep metadata to newer software.

## Rationale & Design Decisions

### Enforced JSON and Core Schema Conformance
By requiring resolved outputs to strictly correspond to the existing `NftType` schema layout definition, this CHIP avoids inventing new token metadata sub-standards. It merely shifts the network retrieval location of standard components from local JSON lines to decentralized URLs, minimizing friction for wallet integrations already equipped to process valid BCMR tokens.

### Evaluation of `commitmentHex` and `bottomAltStackHex`
Allowing the key variable component to evaluate as either the `commitmentHex` or the standard un-hashed execution variable (`bottomAltStackHex`) provides flexibility across different minting and protocol setups. This ensures developers running complex dynamic on-chain logic can map to the identical external type identifier payload seamlessly.

### Placement within IdentitySnapshot Context
This specification deliberately places the `nftsBaseUri` object inside the **`IdentitySnapshot`** extensions block (the timestamped identity state root) rather than inside the nested `TokenCategory` or `NftCategory` sub-schemas. 

This layout configuration is enforced due to the structural boundaries of the current JSON schema definitions in the official BCMR specification:
1. **Existing Extension Anchor:** The `IdentitySnapshot` object natively exposes an open-ended `extensions` definition map intended specifically for vendor-agnostic protocol add-ons. 
