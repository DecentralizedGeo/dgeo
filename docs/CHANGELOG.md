# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [1.2.0] - 2026-01-08

### Added

- **NEW:** `dgeo:cids` - REQUIRED scalar string array for queryable CID discovery via pgstac and STAC API
- **NEW:** `dgeo:piece_cids` - OPTIONAL scalar string array for queryable Filecoin Piece CID (commP) references
- **NEW:** `dgeo:cid` - OPTIONAL asset-level field for explicit CID-to-asset correlation
- **NEW:** Asset-level `dgeo:cid_profile` field for DAG metadata at the asset level
- **NEW:** Asset-level extension scope - dgeo fields can now be used in STAC asset objects
- **NEW:** CID format validation via JSON Schema regex patterns (CIDv0, CIDv1, Piece CID)
- **NEW:** `uniqueItems` and `minItems` constraints on CID arrays
- **NEW:** Queryability support for pgstac and STAC Browser UI
- **NEW:** MIGRATION_GUIDE.md with automated conversion scripts

### Changed

- **BREAKING:** Replaced `dgeo:assets` nested array with flat `dgeo:cids` array for pgstac queryability
- **BREAKING:** Moved CID Profile metadata from `dgeo:assets[].cid_profile` to asset-level `dgeo:cid_profile`
- **BREAKING:** Moved asset roles from `dgeo:assets[].roles` to standard STAC `assets[key].roles`
- **BREAKING:** Moved asset descriptions from `dgeo:assets[].description` to standard STAC `assets[key].description`
- **BREAKING:** Changed asset naming from `dgeo:assets[].name` to standard STAC asset keys
- **BREAKING:** Flattened Piece CIDs from `dgeo:assets[].piece_cid` to `dgeo:piece_cids[]` array
- Updated JSON Schema to JSON Schema Draft-07 with stricter validation
- Improved schema descriptions and field documentation
- Updated all example files (`item.json`, `item-core.json`, `collection.json`, `item-invalid.json`)
- Complete README.md rewrite with queryability documentation, CID-to-asset correlation examples, and migration guidance

### Removed

- **BREAKING:** `dgeo:assets` nested object array (replaced with `dgeo:cids` flat array)
- **BREAKING:** `dgeo:context` field (not needed for MVP; can be reintroduced in future versions)
- **BREAKING:** `dgeo_asset` JSON Schema definition (replaced with `dgeo_asset_fields`)

### Deprecated

- v1.0-wip2 schema is now deprecated. See MIGRATION_GUIDE.md for upgrade instructions.

### Migration Notes

This release represents a **major breaking change** from v1.0-wip2. All existing Items and Collections using the dgeo extension must be migrated. Key changes:

- Replace `dgeo:assets` array with `dgeo:cids` array
- Extract `piece_cid` values into `dgeo:piece_cids` array  
- Move `cid_profile` to asset-level `dgeo:cid_profile`
- Merge roles and descriptions into standard STAC asset fields
- Add `dgeo:cid` to each asset for programmatic correlation
- Remove `dgeo:context` (store externally if needed)

See [MIGRATION_GUIDE.md](./MIGRATION_GUIDE.md) for detailed instructions and automated conversion scripts.

### Fixed

- pgstac queryability issues with nested object arrays
- STAC Browser UI filter compatibility
- CID-to-asset correlation challenges with HTTP gateway URLs

[Unreleased]: <https://github.com/DecentralizedGeo/dgeo-asset/compare/v1.2.0...HEAD>
[1.2.0]: <https://github.com/DecentralizedGeo/dgeo-asset/compare/v1.0.0...v1.2.0>
