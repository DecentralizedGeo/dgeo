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
2. To provide **asset-level DAG metadata** that describes how content-addressed structures were generated, enabling reproducible verification and reconstruction.
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

## Fields

The fields in the table below can be used in these parts of STAC documents:

- [ ] Catalogs  
- [x] Collections  
- [x] Item Properties (including Summaries in Collections)  
- [x] Assets (for both Collections and Items, including Item Asset Definitions in Collections)  
- [ ] Links  

### Properties-Level Fields

| Field Name | Type | Description |
| --- | --- | --- |
| `dgeo:cids` | string[] | **REQUIRED.** Array of IPFS Content Identifiers (CIDs) associated with this Item or Collection. Queryable via pgstac and STAC API. |
| `dgeo:piece_cids` | string[] | OPTIONAL. Array of Filecoin Piece CIDs (commP) used for storage verification and proof-of-replication workflows. Queryable via pgstac and STAC API. |

### Asset-Level Fields

| Field Name | Type | Description |
| --- | --- | --- |
| `dgeo:cid` | string | OPTIONAL. The IPFS CID that this asset represents. MUST appear in the Item's `dgeo:cids` array. Enables programmatic CID-to-asset correlation. |
| `dgeo:cid_profile` | object | OPTIONAL. Technical details about how the CID's DAG was generated (chunking, hashing, layout, sharding). See [CID Profile Object](#cid-profile-object). |

### Field Details

#### `dgeo:cids`

**Type:** Array of strings  
**REQUIRED** when the dgeo extension is declared.

An array of IPFS Content Identifiers (CIDs) associated with this Item or Collection. Each CID **MUST** be immutable; mutable pointers such as IPNS **MUST NOT** be included.

This field is designed for **queryability** via pgstac and STAC API CQL2 filters, enabling users to search for Items by CID.

**CID Format Validation:**

- CIDv0: `^Qm[1-9A-HJ-NP-Za-km-z]{44}$`
- CIDv1: `^b[a-z2-7]{58,}$`

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

**Lookup Pattern:**

```python
def find_asset_by_cid(item, target_cid):
    for asset_key, asset in item['assets'].items():
        if asset.get('dgeo:cid') == target_cid:
            return asset_key, asset
    return None, None
```

#### `dgeo:cid_profile` (Asset-Level)

**Type:** Object  
**OPTIONAL**

Technical details about how the CID's DAG was generated (chunking, hashing, layout, sharding, etc.). This metadata enables reproducible DAG reconstruction and verification workflows.

See the [CID Profile Object](#cid-profile-object) section for detailed field definitions.

## Queryability

The `dgeo` extension is designed for queryability via **pgstac** and **STAC API** implementations that support CQL2 filtering.

### pgstac Compatibility

Both `dgeo:cids` and `dgeo:piece_cids` are **scalar string arrays**, which work seamlessly with pgstac's `a_contains` operator and PostgreSQL's native JSONB array containment (`@>`) operator.

**Example SQL Query:**

```sql
-- Find all Items containing a specific CID
SELECT * FROM pgstac.items 
WHERE content->'properties'->'dgeo:cids' @> '["bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi"]'::jsonb;
```

### CQL2 Query Examples

**Search by CID:**

```json
{
  "filter-lang": "cql2-json",
  "filter": {
    "op": "a_contains",
    "args": [
      {"property": "dgeo:cids"},
      ["bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi"]
    ]
  }
}
```

**Search by Piece CID:**

```json
{
  "filter-lang": "cql2-json",
  "filter": {
    "op": "a_contains",
    "args": [
      {"property": "dgeo:piece_cids"},
      ["baga6ea4seaqao7s73y24kcutaosvacpdjgfe5pw76ooefnyqw4ynr3d2y6x2mpq"]
    ]
  }
}
```

### STAC Browser UI Support

Scalar array fields like `dgeo:cids` will appear in STAC Browser's filter dropdown menus, allowing users to interactively filter Items by CID values.

## CID-to-Asset Correlation

After querying Items by CID, clients often need to identify **which asset** that CID corresponds to. The `dgeo:cid` asset-level field enables this programmatic correlation.

### Why This Matters

Consider an Item with multiple assets:

```json
{
  "properties": {
    "dgeo:cids": ["bafybei111...", "bafybei222...", "bafybei333..."]
  },
  "assets": {
    "red": {"href": "https://gateway.pinata.cloud/ipfs/bafybei111..."},
    "nir": {"href": "https://example.com/nir.tif"},
    "product_bundle": {"href": "ipfs://bafybei222..."}
  }
}
```

**Problem:** After querying for `bafybei111...`, how do you know it corresponds to the `red` asset?

**Solution:** Add `dgeo:cid` to each asset:

```json
{
  "properties": {
    "dgeo:cids": ["bafybei111...", "bafybei222..."]
  },
  "assets": {
    "red": {
      "href": "https://gateway.pinata.cloud/ipfs/bafybei111...",
      "dgeo:cid": "bafybei111...",
      "dgeo:cid_profile": {...}
    },
    "product_bundle": {
      "href": "ipfs://bafybei222...",
      "dgeo:cid": "bafybei222...",
      "dgeo:cid_profile": {...}
    }
  }
}
```

### Implementation Examples

**Python:**

```python
from pystac_client import Client

client = Client.open("https://example.com/stac")

# Search by CID
search = client.search(
    filter={
        "op": "a_contains",
        "args": [{"property": "dgeo:cids"}, ["bafybei111..."]]
    }
)

for item in search.items():
    asset_key, asset = find_asset_by_cid(item, "bafybei111...")
    if asset:
        print(f"Found CID in asset '{asset_key}'")
        print(f"  href: {asset['href']}")
        print(f"  type: {asset['type']}")
```

**JavaScript:**

```javascript
function findAssetByCid(item, targetCid) {
  for (const [assetKey, asset] of Object.entries(item.assets)) {
    if (asset['dgeo:cid'] === targetCid) {
      return { assetKey, asset };
    }
  }
  return null;
}

const items = await fetch('/search', {
  method: 'POST',
  body: JSON.stringify({
    filter: {
      op: 'a_contains',
      args: [{ property: 'dgeo:cids' }, ['bafybei111...']]
    }
  })
}).then(r => r.json());

items.features.forEach(item => {
  const result = findAssetByCid(item, 'bafybei111...');
  if (result) {
    console.log(`Found CID in asset '${result.assetKey}'`);
  }
});
```

## CID Profile Object

The **CID Profile Object** describes the parameters that affect DAG and CID generation, conceptually aligned with UnixFS and IPIP-0499 UnixFS parameters. This allows independent systems to re-chunk or verify DAGs in a reproducible way.

When `cid_profile` is present, the following fields are **RECOMMENDED**: `cid_version`, `chunk_algorithm`, `dag_layout`, and `hash_function`.

| Field Name | Type | Description |
| --- | --- | --- |
| `cid_version` | integer | CID version (`0` or `1`). RECOMMENDED when `cid_profile` is present. [Details](https://docs.ipfs.tech/concepts/content-addressing/#cid-versions) on encoding differences. |
| `multibase_encoding` | string | OPTIONAL. Multibase encoding of the CID string (e.g., `base32`, `base58btc`). |
| `hash_function` | string | RECOMMENDED. Multihash function used when constructing the CID (e.g., `sha2-256`, `blake3`). |
| `chunk_algorithm` | string | RECOMMENDED. Algorithm used to split data (e.g., `fixed`, `balanced`, `rabin`). |
| `chunk_size` | integer | OPTIONAL. Target size of chunks in bytes. |
| `dag_width` | integer | OPTIONAL. Maximum number of links per block. |
| `dag_layout` | string | RECOMMENDED. Layout algorithm (e.g., `balanced`, `trickle`, `hamt-directory`). |
| `directory_wrapping` | boolean | OPTIONAL. Wrap single files in directories to retain filename metadata. |
| `hamt_directory_fanout` | integer | OPTIONAL. Fanout for HAMT-sharded directories (default bitwidth is 8 == 256 leaves). |
| `hamt_directory_threshold` | integer | OPTIONAL. Threshold size at which a regular directory transitions to HAMT-sharded representation. |
| `leaf_envelope` | string | OPTIONAL. How leaf nodes are represented (`raw` or `dag-pb`), enabling validation of expected leaf node types. |

JSON Schema for `cid_profile` uses `"additionalProperties": true` to allow other UnixFS/IPLD parameters in the future.

**Example:**

```json
{
  "dgeo:cid_profile": {
    "cid_version": 1,
    "chunk_algorithm": "fixed",
    "chunk_size": 262144,
    "dag_layout": "balanced",
    "hash_function": "sha2-256"
  }
}
```

## Usage Patterns

### 1. Asset Mirroring

In this pattern, core assets are mirrored on decentralized storage with CIDs tracked at the properties level and asset-level metadata.

**Example:**

```json
{
  "properties": {
    "dgeo:cids": ["bafybei111...", "bafybei222..."]
  },
  "assets": {
    "red": {
      "href": "https://gateway.pinata.cloud/ipfs/bafybei111...",
      "type": "image/tiff",
      "roles": ["data", "mirror"],
      "dgeo:cid": "bafybei111...",
      "dgeo:cid_profile": {
        "cid_version": 1,
        "chunk_algorithm": "fixed",
        "chunk_size": 262144,
        "dag_layout": "balanced",
        "hash_function": "sha2-256"
      }
    },
    "nir": {
      "href": "https://gateway.pinata.cloud/ipfs/bafybei222...",
      "type": "image/tiff",
      "roles": ["data", "mirror"],
      "dgeo:cid": "bafybei222...",
      "dgeo:cid_profile": {
        "cid_version": 1,
        "chunk_algorithm": "fixed",
        "chunk_size": 262144,
        "dag_layout": "balanced",
        "hash_function": "sha2-256"
      }
    }
  }
}
```

### 2. Multiple CID Representations

When a single logical asset has multiple CID representations (different chunking, compression, or layout), create separate asset entries.

**Example:**

```json
{
  "properties": {
    "dgeo:cids": ["bafybei111...", "QmLegacy..."]
  },
  "assets": {
    "visual": {
      "href": "https://gateway.pinata.cloud/ipfs/bafybei111...",
      "type": "image/tiff",
      "roles": ["data", "primary"],
      "description": "Primary CID v1 using fixed chunking",
      "dgeo:cid": "bafybei111...",
      "dgeo:cid_profile": {
        "cid_version": 1,
        "chunk_algorithm": "fixed",
        "chunk_size": 262144
      }
    },
    "visual-legacy": {
      "href": "ipfs://QmLegacy...",
      "type": "image/tiff",
      "roles": ["data", "alternative"],
      "description": "Legacy CID v0 for backward compatibility",
      "dgeo:cid": "QmLegacy...",
      "dgeo:cid_profile": {
        "cid_version": 0,
        "chunk_algorithm": "fixed",
        "chunk_size": 262144
      }
    }
  }
}
```

### 3. Container Resources (CAR Files / Directories)

A `dgeo` resource may represent a container that aggregates multiple assets, such as an IPFS directory or CAR file.

**Example:**

```json
{
  "properties": {
    "dgeo:cids": ["bafybei111...", "bafybei222...", "bafybei333..."],
    "dgeo:piece_cids": ["baga6ea..."]
  },
  "assets": {
    "red": {
      "href": "https://gateway.pinata.cloud/ipfs/bafybei111...",
      "dgeo:cid": "bafybei111...",
      "dgeo:cid_profile": {...}
    },
    "nir": {
      "href": "https://gateway.pinata.cloud/ipfs/bafybei222...",
      "dgeo:cid": "bafybei222...",
      "dgeo:cid_profile": {...}
    },
    "product_bundle": {
      "href": "ipfs://bafybei333...",
      "type": "application/vnd.ipld.car",
      "title": "Complete Product Bundle (CAR)",
      "roles": ["archive"],
      "description": "CAR archive containing all Level-2 product files",
      "dgeo:cid": "bafybei333...",
      "dgeo:cid_profile": {
        "cid_version": 1,
        "dag_layout": "hamt-directory",
        "hash_function": "sha2-256",
        "directory_wrapping": true,
        "hamt_directory_fanout": 256,
        "hamt_directory_threshold": 3
      }
    }
  }
}
```

## Migration from v1.0

**⚠️ Breaking Changes:** Version 1.2 is a major breaking change from v1.0-wip2. Migration is required for existing implementations.

### Key Changes

| v1.0-wip2 | v1.2 | Notes |
| --- | --- | --- |
| `dgeo:assets[].cid` | `dgeo:cids[]` | Flattened to scalar array for queryability |
| `dgeo:assets[].piece_cid` | `dgeo:piece_cids[]` | Flattened to scalar array for queryability |
| `dgeo:assets[].cid_profile` | `assets[key].dgeo:cid_profile` | Moved to asset level |
| `dgeo:assets[].roles` | `assets[key].roles` | Standard STAC asset roles |
| `dgeo:assets[].description` | `assets[key].description` | Standard STAC asset description |
| `dgeo:assets[].name` | Asset object key | Standard STAC pattern |
| N/A | `assets[key].dgeo:cid` | **NEW:** Explicit CID-to-asset correlation |
| `dgeo:context` | **REMOVED** | Not needed for MVP |

### Conversion Example

**v1.0-wip2:**

```json
{
  "properties": {
    "dgeo:assets": [
      {
        "name": "red",
        "cid": "bafybei...",
        "roles": ["mirror", "data"],
        "description": "IPFS mirror of red band",
        "cid_profile": {...}
      }
    ],
    "dgeo:context": {...}
  },
  "assets": {
    "red": {
      "href": "https://example.com/red.tif",
      "roles": ["data"]
    }
  }
}
```

**v1.2:**

```json
{
  "properties": {
    "dgeo:cids": ["bafybei..."]
  },
  "assets": {
    "red": {
      "href": "https://gateway.pinata.cloud/ipfs/bafybei...",
      "roles": ["data", "mirror"],
      "description": "IPFS mirror of red band",
      "dgeo:cid": "bafybei...",
      "dgeo:cid_profile": {...}
    }
  }
}
```

### Migration Checklist

- [ ] Replace `dgeo:assets` array with `dgeo:cids` array
- [ ] Extract all `piece_cid` values into `dgeo:piece_cids` array
- [ ] Move `cid_profile` to asset-level `dgeo:cid_profile`
- [ ] Merge `roles` and `description` into standard asset fields
- [ ] Add `dgeo:cid` to each asset for correlation
- [ ] Remove `dgeo:context` (store externally if needed)
- [ ] Update schema validation to v1.2
- [ ] Update pgstac queryable definitions
- [ ] Update client code that reads `dgeo:assets`
- [ ] Update ingest pipelines
- [ ] Re-index existing Items in STAC API

## Implementation Best Practices

### Synchronization

`dgeo:cids` and asset-level `dgeo:cid` fields **SHOULD** be generated programmatically at ingest/publish time from authoritative sources (storage backends, asset manifest) rather than edited manually.

### Mutability

Mutable pointers (e.g., IPNS) **MUST NOT** be included in `dgeo:cids`. All CIDs must be immutable to ensure reproducible queries and scientific integrity.

### Indexing

STAC APIs using pgstac **SHOULD** register `dgeo:cids` and `dgeo:piece_cids` as queryables to enable CQL2 filtering.

### Asset Correlation

When using HTTP gateway URLs for asset `href` values, always include `dgeo:cid` at the asset level to enable programmatic CID-to-asset correlation.

## Contributing

All contributions are subject to the [STAC Specification Code of Conduct](https://github.com/radiantearth/stac-spec/blob/master/CODE_OF_CONDUCT.md).

For contributions, please follow the [STAC specification contributing guide](https://github.com/radiantearth/stac-spec/blob/master/CONTRIBUTING.md).

### Running tests

1. `npm install`  
2. `npm test`  
