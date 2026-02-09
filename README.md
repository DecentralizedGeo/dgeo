# Decentralized Geospatial Extension Specification

- **Title:** Decentralized Geospatial (dgeo)  
- **Identifier:** <https://raw.githubusercontent.com/DecentralizedGeo/dgeo/refs/heads/main/json-schema/schema.json>
- **Field Name Prefix:** `dgeo`  
- **Scope:** Item, Collection, Assets  
- **Extension Maturity Classification:** Proposal  
- **Owner:** @DecentralizedGeo  
- **Version:** 1.0.0

## Description

The **Decentralized Geospatial (dgeo)** extension provides a standard set of fields to associate STAC Items and Collections with resources on the decentralized web (dweb), such as IPFS, Filecoin, and other content-addressed storage systems.

The primary objectives of this extension are:

1. To enable **queryable CID-based discovery** through STAC API and pgstac, allowing users to search for Items by Content Identifier (CID).
2. To provide **asset-level DAG metadata** that describes how content-addressed structures were generated, enabling reproducible verification and reconstruction. More information on common conventions for DAG creation can be found in [IPIP-499: UnixFS CID Profiles](https://bafybeif6uxug72kmxwqtyfvojuwfp5v4ug27b4zevgwisoupigut6x4exy.ipfs.inbrowser.link/ipips/ipip-0499/).

3. To separate **discovery** (queryable CID arrays) from **access** (asset hrefs and descriptions), following STAC best practices.

Unlike extensions that describe *how* to access a specific file, `dgeo` describes **what** decentralized resources exist in a queryable, pgstac-compatible format.

> **Relation to other extensions**  
>
> - The `dgeo` extension is complementary to the [Alternate Assets extension](https://github.com/stac-extensions/alternate-assets). Alternate Assets provides alternate *location URIs* for the same file, while `dgeo` provides *content-addressed identifiers and technical DAG profiles*.
> - Like the [MLM extension](https://github.com/stac-extensions/mlm), `dgeo` is designed to compose with multiple other extensions. A properly described decentralized dataset will typically use `dgeo` alongside EO, Raster, File, and possibly MLM depending on context.

- Examples:  
  - [Item example](./examples/item.json): Landsat scene with multiple CIDs and asset-level metadata.  
  - [Core example](./examples/item-core.json): Multiple CID representations for the same asset.
  - [Collection example](./examples/collection.json): Collection-level dgeo fields.  
- [JSON Schema](./json-schema/schema.json) (JSON Schema Draft-07).
- [Changelog](./CHANGELOG.md)  

## Scope and usage

The `dgeo` extension **MUST** only be declared in STAC Items and Collections via the `stac_extensions` array.

- **Catalogs**  
  - Catalogs **MUST NOT** declare the `dgeo` extension in `stac_extensions`, but MAY contain child Items or Collections that implement it.

- **Collections**  
  - Collections **MAY** implement `dgeo`.  
  - When used at the Collection level, `dgeo:cids` and `dgeo:piece_cids` describe collection-wide decentralized resources (for example, a collection-level CAR archive or IPFS directory root).
  - Asset-level `dgeo` fields MAY be used in Collection assets.

- **Items**  
  - Items **MAY** implement `dgeo`.  
  - Item-level `dgeo` fields describe decentralized resources specific to that Item (for example, CIDs corresponding to raster assets).

- **Assets**
  - Both Item and Collection assets **MAY** include `dgeo:cid` and `dgeo:cid_profile` fields to provide asset-specific decentralized metadata.

## Field Details

The fields in the table below can be used in these parts of STAC documents:

- [ ] Catalogs  
- [x] Collections 
- [x] Item Properties (including Summaries in Collections)  
- [x] Assets (for both Collections and Items, including Item Asset Definitions in Collections)  
- [ ] Links  

### Property Level Fields

| Field Name | Type | Description |
| --- | --- | --- |
| `dgeo:cids` | string[] | **REQUIRED.** Array of IPFS Content Identifiers (CIDs) associated with this Item or Collection. Queryable via pgstac and STAC API. |
| `dgeo:piece_cids` | string[] | OPTIONAL. Array of Filecoin Piece CIDs (commP) used for storage verification and proof-of-replication workflows. Queryable via pgstac and STAC API. |

#### `dgeo:cids`

**Type:** Array of strings  
**REQUIRED** when the dgeo extension is declared.

An array of IPFS Content Identifiers (CIDs) associated with this Item or Collection. Each CID **MUST** be immutable; mutable pointers such as IPNS **MUST NOT** be included.

This field is designed for **queryability** via pgstac and STAC API CQL2 filters, enabling users to search for Items by CID.

**CID Format Validation:**

- CIDv0: `^Qm[1-9A-HJ-NP-Za-km-z]{44}$` A 46-character string starting with "Qm", base58-encoded multihash
- CIDv1: `^b[a-z2-7]{58,}$` base32-encoded self-describing multiformat protocol

**Constraints:**

- Minimum 1 item (`minItems: 1`)
- All items must be unique (`uniqueItems: true`)

**Example:**

```json
{
  "properties": {
    "datetime": "2020-12-04T22:38:32Z",
    "dgeo:cids": [
      "bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi",
      "bafybeic2zycyt36xnkbvbzqbrmhb3jndcylvqywb4zfqn7i6kcsjjl62ka"
    ]
  }
}
```

#### `dgeo:piece_cids`

**Type:** Array of strings  
**OPTIONAL**

An array of Filecoin Piece CIDs (commP) used for storage verification and proof-of-replication workflows. Like `dgeo:cids`, this field is queryable via pgstac.

**Piece CID Format Validation:**

- `^baga6ea[a-z2-7]{52,}$`

**Constraints:**

- All items must be unique (`uniqueItems: true`)

**Example:**

```json
{
  "properties": {
    "dgeo:cids": ["bafybei..."],
    "dgeo:piece_cids": [
      "baga6ea4seaqao7s73y24kcutaosvacpdjgfe5pw76ooefnyqw4ynr3d2y6x2mpq"
    ]
  }
}
```

### Asset Level Fields

| Field Name | Type | Description |
| --- | --- | --- |
| `dgeo:cid_profile` | object | OPTIONAL. Technical details about how the CID's DAG was generated (chunking, hashing, layout, sharding). See [CID Profile Object](#cid-profile-object). |
| `dgeo:cid` | string | OPTIONAL. The IPFS CID that this asset represents. MUST appear in the Item's `dgeo:cids` array. Enables programmatic CID-to-asset correlation. |
| `dgeo:piece_cid` | string | OPTIONAL. The Filecoin Piece CID (commP) that this specific asset represents. When present, this CID **MUST** also appear in the Item's `dgeo:piece_cids` array at the properties level. |

#### `dgeo:cid_profile` (Asset-Level)

**Type:** Object  
**OPTIONAL**

Technical details about how the CID's DAG was generated (chunking, hashing, layout, sharding, etc.). This metadata enables reproducible DAG reconstruction and verification workflows.

See the [CID Profile Object](#cid-profile-object) section for detailed field definitions.

#### `dgeo:cid` (Asset-Level)

**Type:** String  
**OPTIONAL**

The IPFS CID that this specific asset represents. When present, this CID **MUST** also appear in the Item's `dgeo:cids` array at the properties level.

This field solves the **CID-to-asset correlation problem**: after querying Items by CID, clients can programmatically identify which asset that CID corresponds to, even when the asset `href` uses an HTTP gateway URL.

**Example:**

```json
{
  "properties": {
    "dgeo:cids": ["bafybeigdyrzt..."]
  },
  "assets": {
    "red": {
      "href": "https://gateway.pinata.cloud/ipfs/bafybeigdyrzt...",
      "type": "image/tiff",
      "dgeo:cid": "bafybeigdyrzt...",
      "dgeo:cid_profile": {...}
    }
  }
}
```

#### `dgeo:piece_cid` (Asset-Level)

**Type:** String  
**OPTIONAL**

The Filecoin Piece CID (commP) that this specific asset represents. When present, this CID **MUST** also appear in the Item's `dgeo:piece_cids` array at the properties level.

This field solves the **Piece CID-to-asset correlation problem**: after querying Items by Piece CID, clients can programmatically identify which asset that Piece CID corresponds to, even when the asset `href` uses an HTTP gateway URL.

**Example:**

```json
{
  "properties": {
    "dgeo:piece_cids": ["baga6ea4seaqao7s73y24kcutaosvacpdjgfe5pw76ooefnyqw4ynr3d2y6x2mpq"]
  },
  "assets": {
    "red": {
      "href": "https://gateway.pinata.cloud/ipfs/bafybeigdyrzt...",
      "type": "image/tiff",
      "dgeo:piece_cid": "baga6ea4seaqao7s73y24kcutaosvacpdjgfe5pw76ooefnyqw4ynr3d2y6x2mpq",
      "dgeo:cid_profile": {...}
    }
  }
}
```

## CID Profile Object

The **CID Profile Object** describes the parameters that affect DAG and CID generation, conceptually aligned with UnixFS and [IPIP-0499 UnixFS parameters](https://bafybeif6uxug72kmxwqtyfvojuwfp5v4ug27b4zevgwisoupigut6x4exy.ipfs.inbrowser.link/ipips/ipip-0499/). This allows independent systems to re-chunk or verify DAGs in a reproducible way.

When `cid_profile` is present, the following fields are **RECOMMENDED**: `cid_version`, `chunking_algorithm`, `dag_layout`, and `hash_function`.

| Field Name | Type | Description |
| --- | --- | --- |
| `cid_version` | integer | RECOMMENDED. Content Identifier (CID) version (0 or 1) specifying the format’s structure and encoding. |
| `hash_function` | string | RECOMMENDED. Multihash function to use (e.g., "sha2-256"). |
| `chunking_algorithm` | string | RECOMMENDED. Algorithm used to split files into chunks (e.g., "fixed-size", "rabin"). |
| `chunk_size` | integer | OPTIONAL. Maximum size of each chunk in bytes. |
| `dag_width` | integer | OPTIONAL. Maximum number of children per node in the DAG. |
| `dag_layout` | string | RECOMMENDED. Layout of the DAG (e.g., "balanced", "balanced-packed", "trickle"). |
| `empty_directories` | boolean | OPTIONAL. Whether empty directories are included in the DAG. |
| `hamt_directory_fanout` | string | OPTIONAL. Maximum number of block entries per HAMT directory node (e.g. "256 blocks"). |
| `hamt_directory_threshold` | string | OPTIONAL. The HAMTDirectory threshold determines when a directory converts to a HAMT structure. |
| `hamt_switch_comparison` | string | OPTIONAL. Comparison operators (`>=` or `>`) for switching to a HAMT structure. |
| `leaves` | string | OPTIONAL. Determines whether file data is stored in a dag-pb-wrapped block or as raw bytes. |
| `hidden_entities` | boolean | OPTIONAL. Whether hidden entities (including dot files) are included in the DAG. |
| `symlinks` | string | OPTIONAL. Method for handling symbolic links (e.g., "preserve", "followed", "skipped"). |
| `mode_permissions` | boolean | OPTIONAL. POSIX file permissions included in the DAG. |
| `mod_time` | boolean | OPTIONAL. File modification time included in the DAG. |

JSON Schema for `cid_profile` uses `"additionalProperties": true` to allow other UnixFS/IPLD parameters in the future.

**Example:**

```json
{
  "dgeo:cid_profile": {
    "cid_version": 1,
    "chunking_algorithm": "fixed-size",
    "chunk_size": 262144,
    "dag_layout": "balanced",
    "hash_function": "sha2-256"
  }
}
```

## Implementation Guide

For details on common **Usage Patterns** and best practices, please see the [Implementation Guide](./docs/implementation-guide.md).

## Contributing

All contributions are subject to the
[STAC Specification Code of Conduct](https://github.com/radiantearth/stac-spec/blob/master/CODE_OF_CONDUCT.md).
For contributions, please follow the
[STAC specification contributing guide](https://github.com/radiantearth/stac-spec/blob/master/CONTRIBUTING.md) Instructions
for running tests are copied here for convenience.

### Running tests

The same checks that run as checks on PR's are part of the repository and can be run locally to verify that changes are valid. 
To run tests locally, you'll need `npm`, which is a standard part of any [node.js installation](https://nodejs.org/en/download/).

First you'll need to install everything with npm once. Just navigate to the root of this repository and on 
your command line run:
```bash
npm install
```

Then to check markdown formatting and test the examples against the JSON schema, you can run:
```bash
npm test
```

This will spit out the same texts that you see online, and you can then go and fix your markdown or examples.

If the tests reveal formatting problems with the examples, you can fix them with:
```bash
npm run format-examples
```
