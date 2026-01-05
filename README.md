# Decentralized Geospatial Extension Specification

- **Title:** Decentralized Geospatial (dgeo)  
- **Identifier:** <https://github.com/DecentralizedGeo/dgeo-asset/blob/main/json-schema/schema.json>  
- **Field Name Prefix:** `dgeo`  
- **Scope:** Item, Collection  
- **Extension Maturity Classification:** Proposal  
- **Owner:** @DecentralizedGeo  

## Description

The **Decentralized Geospatial (dgeo)** extension provides a standard set of fields to associate STAC Items and Collections with resources on the decentralized web (dweb), such as IPFS, Filecoin, and other decentralized storage backends.

The primary objectives of this extension are:

1. To enable building **dweb-aware collections and items** that can be searched using [Content Identifiers](https://docs.ipfs.tech/concepts/content-addressing/) (CID), an addressable "label" based on the content itself (e.g. asset data associated with the Item).
2. To separate **discovery** of decentralized resources from concrete **access protocols**, allowing clients to plug in different dweb stacks without changing core STAC structures.
3. To record, when desired, the **technical profile** of content-addressed DAGs (chunking, layout, hashing, sharding) so that clients can verify or reconstruct structures in a reproducible way.

Unlike extensions that describe *how* to access a specific file (for example, Storage or File extensions), `dgeo` describes **what** decentralized resources exist, how they relate to core STAC assets and containers, and how their DAGs are constructed.

> **Relation to other extensions**  
>
> - The `dgeo` extension is complementary to the [Alternate Assets extension](https://github.com/stac-extensions/alternate-assets). Alternate Assets focuses on alternate *location URIs* for the same file, while `dgeo` focuses on *content-addressed identifiers and DAG profiles*.
> - Like the [MLM extension](https://github.com/stac-extensions/mlm), `dgeo` is designed to compose with multiple other extensions. A properly described decentralized dataset or model will typically use `dgeo` alongside EO, Raster, File, and possibly MLM depending on context.

- Examples:  
  - [Item example](./examples/item.json): Basic usage with `dgeo:assets` on an Item.  
  - [Collection example](./examples/collection.json): Usage of `dgeo` at the Collection level.  
- [JSON Schema](./json-schema/schema.json) (JSON Schema Draft-07).
- [Changelog](./CHANGELOG.md)  

## Scope and usage

The `dgeo` extension **MUST** only be declared in STAC Items and Collections via the `stac_extensions` array.

- **Catalogs**  
  - Catalogs **MUST NOT** declare the `dgeo` extension in `stac_extensions`, but MAY contain child Items or Collections that implement it.

- **Collections**  
  - Collections **MAY** implement `dgeo`.  
  - When used at the Collection level, `dgeo:assets` and `dgeo:context` describe **default behavior, summaries, or collection-wide decentralized resources** (for example, a collection-level [Content Addressable Archive](https://ipld.io/specs/transport/car/) (CAR) or IPFS path to a directory) and MAY be echoed or specialized at the Item level.

- **Items**  
  - Items **MAY** implement `dgeo`.  
  - Item-level `dgeo` fields describe decentralized resources specific to that Item (for example, CIDs corresponding to a particular raster asset).

The `dgeo` extension **does not** introduce fields scoped directly to Assets, Catalogs, or Links. Instead, it defines Item/Collection properties that **reference** decentralized representations of logical assets, directories, or preserved data on Filecoin.

## Fields

The fields in the table below can be used in these parts of STAC documents:

- [ ] Catalogs  
- [x] Collections  
- [x] Item Properties (including Summaries in Collections)  
- [ ] Assets (for both Collections and Items, including Item Asset Definitions in Collections)  
- [ ] Links  

| Field Name      | Type                                          | Description |
| --------------- | --------------------------------------------- | ----------- |
| `dgeo:assets` | [[Assets Object](#dgeoassets-resource-map)] | **RECOMMENDED.** A discovery-layer map of decentralized resources associated with the Item or Collection. |
| `dgeo:context` | [Context Object](#dgeocontext-user-flexibility) | A reserved bucket for implementation-specific metadata and operational context. |

Implementations **SHOULD** treat `dgeo:assets` as a **derivative, discovery-oriented index** and **MUST NOT** use it as a replacement for core `assets`.

## `dgeo:assets` resource map

**Type:** Array of Objects (each object is a *dgeo Asset Resource*).

`dgeo:assets` describes decentralized resources that correspond to:  

- Individual logical assets in the Item’s `assets` object (for example, a GeoTIFF retrievable from IPFS).  
- Container-level resources that group multiple logical assets (for example, an IPFS directory, CAR archive representing a full product bundle, or content prepared and preserved on Filecoin).

`dgeo:assets` **MUST NOT** enumerate block-level CIDs or internal DAG nodes. For complex DAGs or long block lists, implementations **SHOULD** reference the root dag of a higher-level CAR, manifest, or directory CID instead.

> **Note:** The `name` field is **not required to be unique** within `dgeo:assets`. Multiple entries with the same `name` MAY exist to represent different DAG structures or backends for the same logical asset (for example, streaming vs archival). Implementations grouping entries **SHOULD** consider both `name` and `roles`.

### dgeo Asset Resource fields

| Field Name      | Type                        | Description |
| --------------- | --------------------------- | ----------- |
| `name`          | string                      | **REQUIRED.** A label for the resource. When the resource represents a decentralized form of a core asset, `name` SHOULD match the corresponding key in the Item’s `assets` object. For container-level resources, it MAY be a stable semantic label such as `collection_root` or `archive`. |
| `cid`           | string                      | **REQUIRED.** The primary content-addressed identifier (for example, IPFS CID). The target **MUST** be immutable; mutable pointers such as IPNS or equivalent MUST be signaled using the `roles` field (see below). |
| `roles`         | string[]                    | Semantic roles describing the purpose of this resource (for example, `["mirror", "data"]`, `["data", "primary"]`, `["collection", "archive"]`, `["mutable"]`). A small, implementation-consistent vocabulary is **RECOMMENDED**; additional roles MAY be added as needed. |
| `description` | string                      | OPTIONAL. Human-readable description of the resource and its relationship to the core assets or collection (for example, “IPFS CAR archive for full Level-2A product”). |
| `piece_cid`     | string                      | OPTIONAL. Filecoin Piece CID (commP) used for storage verification and proof-of-replication workflows. |
| `cid_profile` | [CID Profile Object](#cid-profile-object) | OPTIONAL. Technical details about how the CID’s DAG was generated (chunking, hashing, layout, sharding, etc.). |

### Recommended `roles` semantics

The following roles are **RECOMMENDED** and SHOULD NOT be reinterpreted with conflicting semantics, similar to how MLM recommends task/role vocabularies.

- `mirror`: Resource mirrors a core `assets` entry via a decentralized backend.  
- `data`: Resource contains primary data content (raster, vector, point cloud, etc.).  
- `primary`: Preferred decentralized representation for general use.  
- `alternative`: Alternate representation (different chunking, compression, or layout).  
- `collection`: Resource represents a container of multiple assets (for example, directory, bundle, or manifest).  
- `archive`: Resource represents an archival or bundled form (for example, CAR, tar in IPFS).  
- `mutable`: Resource is intentionally mutable (for example, IPNS-style pointer). Consumers SHOULD NOT treat it as reproducible.

Implementations MAY add additional roles, but SHOULD avoid redefining the meaning of these core roles.

## CID Profile Object

The **CID Profile Object** describes the parameters that affect DAG and CID generation, conceptually aligned with UnixFS and IPIP-0499 UnixFS parameters. This allows independent systems to re-chunk or verify DAGs in a reproducible way.

When `cid_profile` is present, the following fields are **RECOMMENDED**: `cid_version`, `chunk_algorithm`, `dag_layout`, and `hash_function`.

| Field Name                   | Type    | Description |
| ---------------------------- | ------- | ----------- |
| `cid_version`                | integer | CID version (for example, `0` or `1`). RECOMMENDED when `cid_profile` is present.  [Details](https://docs.ipfs.tech/concepts/content-addressing/#cid-versions) on encoding difference. |
| `multibase_encoding`         | string  | OPTIONAL. Multibase encoding of the CID string (for example, `base32`). |
| `hash_function`              | string  | RECOMMENDED. Multihash function used when constructing the CID (e.g. `sha2-256`, `blake3`). |
| `chunk_algorithm`            | string  | RECOMMENDED: Algorithm used to split data (for example, `fixed`, `balanced`, `rabin`). |
| `chunk_size`                 | integer | OPTIONAL. Target size of chunks in bytes. |
| `dag_width`                  | integer | OPTIONAL. Maximum number of links per block. |
| `dag_layout`                 | string  | RECOMMENDED. Layout algorithm (for example, `balanced`, `trickle`, `hamt-directory`). |
| `directory_wrapping`         | boolean | OPTIONAL. Option to encode content by wrapping for single files with `Directories` as to retain the name of a single file. |
| `hamt_directory_fanout`      | integer | OPTIONAL. Fanout determines how many "buckets" a directory is split into for HAMT-based UnixFS directories (default bitwidth is 8 == 256 leaves). |
| `hamt_directory_threshold` | integer | OPTIONAL. Threshold size at which a regular directory transitions to a HAMT-sharded representation. |
| `leaf_envelope`              | string  | OPTIONAL. How leaf nodes are represented, either `raw` or `dag-pb`, enabling validation of expected leaf node types. |

JSON Schema for `cid_profile` SHOULD:

- Declare explicit types for the fields above.  
- Use `"additionalProperties": true` to allow other UnixFS/IPLD parameters in the future.  

## `dgeo:context` user flexibility

**Type:** Object  

`dgeo:context` is a reserved bucket for implementation-specific logic, policies, and operational metadata that do not belong in the core `dgeo:assets` model but are still relevant for decentralized geospatial workflows.

Similar to MLM’s `mlm:hyperparameters` and other open objects, this is an **open, implementation-defined** structure. Keys and values inside `dgeo:context` are not globally standardized and **MUST NOT** affect cross-implementation validation semantics directly. Fields that become widely used SHOULD be proposed as new core `dgeo:` fields.

### Conventions

Implementations **SHOULD** follow these conventions:

- Keys **SHOULD** be vendor or organization prefixed and use `snake_case`, for example: `myorg_pinning_policy`, `myorg_gateway_preferences`.
- Values MAY be any JSON type (object, array, string, number, boolean, null).  
- Keys SHOULD be stable and documented within the organization, especially if they appear in public catalogs.  

JSON Schema for `dgeo:context` **MUST** set `"additionalProperties": true`, but MAY include a `patternProperties` hint to encourage `snake_case` keys.

### Example

```json
"dgeo:context": {
  "myorg_pinning_policy": {
    "preferred_gateways": [
      "https://ipfs.myorg.org",
      "https://dweb.link"
    ],
    "replication_factor": 3,
    "automatic_pin": true
  },
  "myorg_verification": {
    "require_cid_profile": true,
    "allowed_hash_functions": ["sha2-256", "blake3"]
  }
}
```

## Usage patterns

The `dgeo` extension enables several repeatable patterns, analogous to how MLM defines patterns for model assets and runtimes.

### 1. Asset mirroring (“search index”)

In this pattern, each asset in the Item’s `assets` object is mirrored by one or more `dgeo:assets` entries.

- `name` SHOULD match the asset key (for example, `B01`, `thumbnail`, `mlm:model`, `xml`).  
- `roles` SHOULD include `["mirror", "data"]` for primary data mirrors.
- `cid_profile` MAY be used for reproducible mirrors of large products.  

### 2. Multi-DAG representations

A single logical asset (for example, a model, a raster, or a bundle) may be available as multiple CIDs optimized for different trade-offs.

- All entries use the same `name`.  
- `roles` distinguish them, for example:  
  - `["data", "primary"]` for the main representation.  
  - `["data", "alternative"]` for an alternate layout or compression.  
- `cid_profile` SHOULD capture differences in chunking, hash function, or layout between versions.  

### 3. Container reference (“folder” / archive)

A `dgeo` resource may represent a container that aggregates multiple assets, such as:  

- IPFS directory containing all item files.  
- CAR file representing a product bundle.  

Typical configuration:

- `name`: a semantic label such as `collection_root` or `bundle`.  
- `roles`: SHOULD include `["collection", "archive"]` or similar.  
- `cid_profile`: SHOULD describe directory layout and any HAMT parameters if applicable.  

### 4. Decentralized model bundles (with MLM)

When used alongside the MLM extension, `dgeo` can describe decentralized packaging of model assets and runtimes.

- `assets["model"]` (with `mlm:model` role) may have one or more `dgeo:assets` entries, with `name: "model"` and roles such as `["mirror", "data"]` or `["data", "primary"]`.  
- A collection-level CAR file containing model, source code, and container images may be represented as `name: "mlm_bundle"`, `roles: ["collection", "archive"]`.  
- `dgeo:context` MAY include runtime-specific hints (for example, gateways preferred for pulling the model artifact).  

## Implementation pitfalls and best practices

### Synchronization drift

- **Risk:** `dgeo:assets` describes resources that often correspond to `assets`. If `assets` are updated or renamed without updating `dgeo:assets`, decentralized references become stale.
- **Best Practice:** `dgeo:assets` SHOULD be generated or updated **programmatically** at ingest/publish time from authoritative sources (assets and storage backends) rather than edited manually.

### Mutability vs reproducibility

- **Risk:** Mutable pointers (for example, IPNS or equivalent) conflict with STAC’s emphasis on reproducible Items if used without explicit signaling or change tracking.
- **Best Practices:**  
  - If the CID is mutable, include the `"mutable"` value in `roles` to signal non-reproducible references.  
  - Track pointer updates in a separate audit log or versioned collection when reproducibility is required.  
  - For scientific or regulatory workflows, clients SHOULD prefer immutable `cid` entries without `"mutable"` semantics.

### Indexing and API behavior

- **Risk:** Exposing many CIDs directly in filters or summaries can hurt index size and query performance.
- **Best Practices:**  
  - STAC APIs SHOULD index and summarize **structural fields** (`name`, `roles`) rather than `cid` values, similar to how MLM and SAT summarize key properties.
  - Collections MAY provide summaries for `dgeo:assets.roles` and `dgeo:assets.name` to support common discovery patterns (for example, “Items with IPFS archives of the full bundle”)

## Contributing

All contributions are subject to the [STAC Specification Code of Conduct](https://github.com/radiantearth/stac-spec/blob/master/CODE_OF_CONDUCT.md).

For contributions, please follow the [STAC specification contributing guide](https://github.com/radiantearth/stac-spec/blob/master/CONTRIBUTING.md).

### Running tests

1. `npm install`  
2. `npm test`  
