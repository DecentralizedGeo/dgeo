# Implementation Guide

This guide provides detailed instructions and patterns for implementing the Decentralized Geospatial (dgeo) extension.

## Table of Contents

1. [Queryability](#queryability)
2. [CID-to-Asset Correlation](#cid-to-asset-correlation)
3. [Usage Patterns](#usage-patterns)
4. [Implementation Best Practices](#implementation-best-practices)

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

### Example Code Use Cases

**Lookup Pattern:**

```python
def find_asset_by_cid(item, target_cid):
    for asset_key, asset in item['assets'].items():
        if asset.get('dgeo:cid') == target_cid:
            return asset_key, asset
    return None, None
```

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
    "red": {"href": "https://ipfs.io/ipfs/bafybei111..."},
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
      "href": "https://ipfs.io/ipfs/bafybei111...",
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
      "href": "https://ipfs.io/ipfs/bafybei111...",
      "type": "image/tiff",
      "roles": ["data", "mirror"],
      "dgeo:cid": "bafybei111...",
      "dgeo:cid_profile": {
        "cid_version": 1,
        "chunking_algorithm": "fixed-size",
        "chunk_size": 262144,
        "dag_layout": "balanced",
        "hash_function": "sha2-256"
      }
    },
    "nir": {
      "href": "https://ipfs.io/ipfs/bafybei222...",
      "type": "image/tiff",
      "roles": ["data", "mirror"],
      "dgeo:cid": "bafybei222...",
      "dgeo:cid_profile": {
        "cid_version": 1,
        "chunking_algorithm": "fixed-size",
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
      "href": "https://ipfs.io/ipfs/bafybei111...",
      "type": "image/tiff",
      "roles": ["data", "primary"],
      "description": "Primary CID v1 using fixed chunking",
      "dgeo:cid": "bafybei111...",
      "dgeo:cid_profile": {
        "cid_version": 1,
        "chunking_algorithm": "fixed-size",
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
        "chunking_algorithm": "fixed-size",
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
      "href": "https://ipfs.io/ipfs/bafybei111...",
      "dgeo:cid": "bafybei111...",
      "dgeo:cid_profile": {...}
    },
    "nir": {
      "href": "https://ipfs.io/ipfs/bafybei222...",
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
        "dag_layout": "trickle",
        "hash_function": "sha2-256",
        "empty_directories": true,
        "hamt_directory_fanout": "256 blocks",
        "hamt_directory_threshold": "256 KiB (links-bytes)"
      }
    }
  }
}
```

## Implementation Best Practices

### Synchronization

`dgeo:cids` and asset-level `dgeo:cid` fields **SHOULD** be generated programmatically at ingest/publish time from authoritative sources (storage backends, asset manifest) rather than edited manually.

### Mutability

Mutable pointers (e.g., IPNS) **MUST NOT** be included in `dgeo:cids`. All CIDs must be immutable to ensure reproducible queries and scientific integrity.

### Indexing

STAC APIs using pgstac **SHOULD** register `dgeo:cids` and `dgeo:piece_cids` as queryables to enable CQL2 filtering.

### Asset Correlation

When using HTTP gateway URLs for asset `href` values, always include `dgeo:cid` at the asset level to enable programmatic CID-to-asset correlation.
