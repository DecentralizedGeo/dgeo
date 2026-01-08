# Migration Guide: dgeo v1.0-wip2 → v1.2

**Version:** 1.2.0  
**Date:** January 8, 2026  
**Status:** Complete

---

## Overview

Version 1.2 of the dgeo STAC extension represents a **major breaking change** from v1.0-wip2. This guide provides step-by-step instructions for migrating existing Items and Collections.

### Why the Breaking Change?

The v1.0-wip2 nested object array structure (`dgeo:assets`) was incompatible with:
- **pgstac queryables** (CQL2 `a_contains` queries)
- **STAC Browser UI** (array filters)
- **PostgreSQL JSONB indexing** (efficient CID lookups)

Version 1.2 flattens the structure to use scalar arrays at the properties level, enabling full queryability while moving descriptive metadata to standard STAC asset objects.

---

## Field Mapping Reference

| v1.0-wip2 Location | v1.2 Location | Type | Notes |
| --- | --- | --- | --- |
| `dgeo:assets[].cid` | `dgeo:cids[]` | string | Queryable array |
| `dgeo:assets[].piece_cid` | `dgeo:piece_cids[]` | string | Queryable array |
| `dgeo:assets[].cid` | `assets[key].dgeo:cid` | string | Correlation field |
| `dgeo:assets[].cid_profile` | `assets[key].dgeo:cid_profile` | object | Asset-level |
| `dgeo:assets[].roles` | `assets[key].roles` | array | Standard STAC |
| `dgeo:assets[].name` | `assets[key]` (key) | - | Asset key |
| `dgeo:assets[].description` | `assets[key].description` | string | Standard STAC |
| `dgeo:context` | **REMOVED** | - | Not needed for MVP |

---

## Migration Steps

### Step 1: Extract CIDs

Collect all `cid` values from `dgeo:assets` into a flat array:

**Before:**
```json
"dgeo:assets": [
  {"name": "red", "cid": "bafybei111..."},
  {"name": "nir", "cid": "bafybei222..."}
]
```

**After:**
```json
"dgeo:cids": [
  "bafybei111...",
  "bafybei222..."
]
```

### Step 2: Extract Piece CIDs

Collect all `piece_cid` values into a separate array:

**Before:**
```json
"dgeo:assets": [
  {
    "name": "bundle",
    "cid": "bafybei333...",
    "piece_cid": "baga6ea..."
  }
]
```

**After:**
```json
"dgeo:cids": ["bafybei333..."],
"dgeo:piece_cids": ["baga6ea..."]
```

### Step 3: Move Metadata to Assets

For each `dgeo:assets` entry, find or create the corresponding asset and merge metadata:

**Before:**
```json
"dgeo:assets": [
  {
    "name": "red",
    "cid": "bafybei111...",
    "roles": ["mirror", "data"],
    "description": "IPFS mirror of red band",
    "cid_profile": {
      "cid_version": 1,
      "chunk_algorithm": "fixed"
    }
  }
],
"assets": {
  "red": {
    "href": "https://example.com/red.tif",
    "type": "image/tiff",
    "roles": ["data"]
  }
}
```

**After:**
```json
"dgeo:cids": ["bafybei111..."],
"assets": {
  "red": {
    "href": "https://gateway.pinata.cloud/ipfs/bafybei111...",
    "type": "image/tiff",
    "roles": ["data", "mirror"],
    "description": "IPFS mirror of red band",
    "dgeo:cid": "bafybei111...",
    "dgeo:cid_profile": {
      "cid_version": 1,
      "chunk_algorithm": "fixed"
    }
  }
}
```

### Step 4: Remove dgeo:context

Remove the `dgeo:context` field. If the context data is still needed, store it externally or in a custom field outside the dgeo extension.

**Before:**
```json
"dgeo:context": {
  "landsat_pinning_policy": {
    "min_replication": 3
  }
}
```

**After:**
```json
// Removed - store externally if needed
```

### Step 5: Validate

Validate the migrated Item/Collection against the v1.2 schema:

```bash
npm run check-examples
```

---

## Automated Conversion Script

### Python

```python
import copy

def migrate_item_v1_to_v1_2(item_v1):
    """Convert dgeo v1.0-wip2 Item to v1.2"""
    item_v1_2 = copy.deepcopy(item_v1)
    
    # Extract dgeo:assets array
    dgeo_assets = item_v1_2['properties'].pop('dgeo:assets', [])
    
    # Initialize new fields
    item_v1_2['properties']['dgeo:cids'] = []
    item_v1_2['properties']['dgeo:piece_cids'] = []
    
    # Process each dgeo asset
    for dgeo_asset in dgeo_assets:
        name = dgeo_asset['name']
        cid = dgeo_asset['cid']
        
        # Add CID to dgeo:cids array
        item_v1_2['properties']['dgeo:cids'].append(cid)
        
        # Add piece_cid if present
        if 'piece_cid' in dgeo_asset:
            item_v1_2['properties']['dgeo:piece_cids'].append(dgeo_asset['piece_cid'])
        
        # Find or create corresponding asset
        if name not in item_v1_2['assets']:
            # Create new asset for container-level resources
            item_v1_2['assets'][name] = {
                'href': f'ipfs://{cid}',
                'type': 'application/octet-stream',
                'roles': dgeo_asset.get('roles', ['data'])
            }
        
        # Merge metadata into asset
        asset = item_v1_2['assets'][name]
        
        # Add dgeo:cid for correlation
        asset['dgeo:cid'] = cid
        
        # Merge roles
        if 'roles' in dgeo_asset:
            existing_roles = set(asset.get('roles', []))
            new_roles = set(dgeo_asset['roles'])
            asset['roles'] = list(existing_roles | new_roles)
        
        # Add description if present
        if 'description' in dgeo_asset:
            asset['description'] = dgeo_asset['description']
        
        # Move cid_profile to asset level
        if 'cid_profile' in dgeo_asset:
            asset['dgeo:cid_profile'] = dgeo_asset['cid_profile']
    
    # Remove dgeo:context if present
    item_v1_2['properties'].pop('dgeo:context', None)
    
    # Remove empty piece_cids array
    if not item_v1_2['properties']['dgeo:piece_cids']:
        del item_v1_2['properties']['dgeo:piece_cids']
    
    return item_v1_2

# Usage
import json

with open('item_v1.json') as f:
    item_v1 = json.load(f)

item_v1_2 = migrate_item_v1_to_v1_2(item_v1)

with open('item_v1_2.json', 'w') as f:
    json.dump(item_v1_2, f, indent=2)
```

### JavaScript

```javascript
function migrateItemV1ToV1_2(itemV1) {
  const itemV1_2 = JSON.parse(JSON.stringify(itemV1)); // Deep clone
  
  // Extract dgeo:assets array
  const dgeoAssets = itemV1_2.properties['dgeo:assets'] || [];
  delete itemV1_2.properties['dgeo:assets'];
  
  // Initialize new fields
  itemV1_2.properties['dgeo:cids'] = [];
  itemV1_2.properties['dgeo:piece_cids'] = [];
  
  // Process each dgeo asset
  for (const dgeoAsset of dgeoAssets) {
    const name = dgeoAsset.name;
    const cid = dgeoAsset.cid;
    
    // Add CID to dgeo:cids array
    itemV1_2.properties['dgeo:cids'].push(cid);
    
    // Add piece_cid if present
    if (dgeoAsset.piece_cid) {
      itemV1_2.properties['dgeo:piece_cids'].push(dgeoAsset.piece_cid);
    }
    
    // Find or create corresponding asset
    if (!itemV1_2.assets[name]) {
      itemV1_2.assets[name] = {
        href: `ipfs://${cid}`,
        type: 'application/octet-stream',
        roles: dgeoAsset.roles || ['data']
      };
    }
    
    // Merge metadata into asset
    const asset = itemV1_2.assets[name];
    
    // Add dgeo:cid for correlation
    asset['dgeo:cid'] = cid;
    
    // Merge roles
    if (dgeoAsset.roles) {
      const existingRoles = new Set(asset.roles || []);
      dgeoAsset.roles.forEach(role => existingRoles.add(role));
      asset.roles = Array.from(existingRoles);
    }
    
    // Add description if present
    if (dgeoAsset.description) {
      asset.description = dgeoAsset.description;
    }
    
    // Move cid_profile to asset level
    if (dgeoAsset.cid_profile) {
      asset['dgeo:cid_profile'] = dgeoAsset.cid_profile;
    }
  }
  
  // Remove dgeo:context if present
  delete itemV1_2.properties['dgeo:context'];
  
  // Remove empty piece_cids array
  if (itemV1_2.properties['dgeo:piece_cids'].length === 0) {
    delete itemV1_2.properties['dgeo:piece_cids'];
  }
  
  return itemV1_2;
}

// Usage
const fs = require('fs');

const itemV1 = JSON.parse(fs.readFileSync('item_v1.json', 'utf8'));
const itemV1_2 = migrateItemV1ToV1_2(itemV1);

fs.writeFileSync('item_v1_2.json', JSON.stringify(itemV1_2, null, 2));
```

---

## Edge Cases

### Multiple CIDs for the Same Asset

**v1.0-wip2 Approach:**
```json
"dgeo:assets": [
  {"name": "visual", "cid": "bafybei111...", "roles": ["primary"]},
  {"name": "visual", "cid": "QmLegacy...", "roles": ["alternative"]}
]
```

**v1.2 Approach:** Create separate asset entries:
```json
"dgeo:cids": ["bafybei111...", "QmLegacy..."],
"assets": {
  "visual": {
    "href": "ipfs://bafybei111...",
    "roles": ["data", "primary"],
    "dgeo:cid": "bafybei111...",
    "dgeo:cid_profile": {...}
  },
  "visual-legacy": {
    "href": "ipfs://QmLegacy...",
    "roles": ["data", "alternative"],
    "dgeo:cid": "QmLegacy...",
    "dgeo:cid_profile": {...}
  }
}
```

### Container-Level Resources

**v1.0-wip2 Approach:**
```json
"dgeo:assets": [
  {
    "name": "product_bundle",
    "cid": "bafybei333...",
    "roles": ["collection", "archive"]
  }
]
```

**v1.2 Approach:** Create a dedicated asset:
```json
"dgeo:cids": ["bafybei333..."],
"assets": {
  "product_bundle": {
    "href": "ipfs://bafybei333...",
    "type": "application/vnd.ipld.car",
    "title": "Complete Product Bundle",
    "roles": ["archive"],
    "dgeo:cid": "bafybei333...",
    "dgeo:cid_profile": {...}
  }
}
```

---

## Post-Migration Tasks

- [ ] Update STAC API queryable definitions
- [ ] Re-index Items in pgstac/stac-fastapi
- [ ] Update client applications
- [ ] Update ingest pipelines
- [ ] Test CQL2 queries
- [ ] Verify STAC Browser UI filters

---

## Validation

After migration, verify that:

1. ✅ All examples validate against the v1.2 schema
2. ✅ `dgeo:cids` is present and non-empty
3. ✅ All `dgeo:cid` values appear in `dgeo:cids`
4. ✅ CID formats match regex patterns
5. ✅ Asset-level metadata is properly merged
6. ✅ No `dgeo:assets` or `dgeo:context` fields remain

---

## Rollback Plan

If migration issues arise:

1. Keep backups of v1.0-wip2 Items
2. Maintain parallel v1.0 and v1.2 endpoints during transition
3. Use feature flags to toggle extension versions
4. Document any known incompatibilities

---

## Support

For questions or issues:
- Open an issue in the [dgeo-asset repository](https://github.com/DecentralizedGeo/dgeo-asset)
- Review the [IMPLEMENTATION_PLAN.md](./docs/spec-design/version-1.2-pgstac-redesign/IMPLEMENTATION_PLAN.md)
- Check the [CID_CORRELATION_SOLUTION.md](./docs/spec-design/version-1.2-pgstac-redesign/CID_CORRELATION_SOLUTION.md)
