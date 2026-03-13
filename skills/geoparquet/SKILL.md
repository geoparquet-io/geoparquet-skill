---
name: geoparquet
description: Work with GeoParquet files - the cloud-native format for geospatial vector data. Covers best practices for creating, optimizing, and distributing GeoParquet, using gpio CLI and DuckDB.
---

# GeoParquet Skill

You are an expert in GeoParquet, the cloud-native columnar format for geospatial vector data. Help users create, optimize, validate, and distribute GeoParquet files following official best practices.

## What is GeoParquet?

GeoParquet is Apache Parquet with standardized geospatial metadata. It combines Parquet's columnar efficiency with proper geometry encoding (WKB), coordinate reference system metadata, and bounding box optimizations.

**Key advantages over legacy formats:**
- **Cloud-native**: Efficient partial reads via HTTP range requests
- **Columnar**: Read only the columns you need
- **Compressed**: zstd compression typically achieves 5-10x reduction
- **Typed**: Strong schema with geometry type enforcement
- **Indexed**: Bbox covering enables fast spatial queries

---

## Best Practices for Distributing GeoParquet

Follow the [OGC GeoParquet distribution guidelines](https://github.com/opengeospatial/geoparquet/blob/main/format-specs/distributing-geoparquet.md):

### Compression

**Use zstd compression at level 15** for distribution. Higher levels (17+) require significantly more computation for <1% additional compression. zstd achieves better compression ratios than snappy with similar decompression speeds.

```bash
# gpio uses zstd by default (level 3 for speed)
# For distribution, use higher compression:
gpio convert geoparquet input.geojson output.parquet --compression-level 15
```

### Spatial Ordering (Critical)

**Data must be spatially sorted** for bbox filtering to work efficiently. Without spatial ordering, queries must scan the entire file even with bbox metadata.

Options for spatial sorting:
- **Hilbert curve** (best): Clusters nearby features together optimally
- **Quadkey/S2/GeoHash**: Grid-based alternatives
- **SORT_BY_BBOX**: GDAL's simpler approach

```bash
# gpio applies Hilbert sorting by default
gpio convert geoparquet input.geojson output.parquet

# Explicit Hilbert sort
gpio sort hilbert input.parquet output.parquet
```

### Bbox Column + Covering Metadata

**Include a bbox column** with GeoParquet 1.1 covering metadata. This struct of four floats (xmin, ymin, xmax, ymax) enables row-group-level spatial filtering.

```bash
# Add bbox column
gpio add bbox input.parquet output.parquet

# Add covering metadata to existing bbox column
gpio add bbox-metadata file.parquet
```

### Row Group Size

**Target 50,000-150,000 rows per group** (or 128-256MB). This balances:
- Smaller groups: Better spatial query latency
- Larger groups: Better full-scan analytics performance

```bash
gpio convert geoparquet input.geojson output.parquet --row-group-size 100000
```

### Partitioning Large Datasets

**Partition datasets >2GB** into multiple spatially-organized files:
- KD-tree: Balanced splits by feature count
- Admin boundaries: Country/region based
- S2/Quadkey/GeoHash: Grid-based

```bash
# KD-tree partitioning (recommended for balanced files)
gpio partition kdtree input.parquet output_dir/ --max-rows-per-file 500000

# Admin boundary partitioning
gpio partition admin input.parquet output_dir/

# Quadkey grid partitioning
gpio partition quadkey input.parquet output_dir/ --resolution 6
```

### STAC Metadata

Use STAC (SpatioTemporal Asset Catalog) for discovery. Media type: `application/vnd.apache.parquet`

```bash
# Single file
gpio publish stac data.parquet data.stac.json

# Partitioned directory (creates Collection + Items)
gpio publish stac ./partitioned/ ./partitioned/ --collection-id "my-dataset"
```

---

## Tools

### gpio (geoparquet-io) - Preferred

**Always prefer gpio** for GeoParquet operations. It implements all best practices by default.

**Installation:**
```bash
# Isolated install (recommended)
pipx install geoparquet-io

# Or with pip/uv
pip install geoparquet-io
uv pip install geoparquet-io

# Verify
gpio --version
```

If gpio is not installed, help the user install it before proceeding.

**Why gpio over alternatives:**

| Feature | gpio | GeoPandas | GDAL/ogr2ogr |
|---------|------|-----------|--------------|
| GeoParquet 1.1 spec | Full | Partial | Partial |
| Hilbert sorting | Default | No | No |
| bbox + covering metadata | Automatic | Manual | No |
| zstd compression | Default | No | No |
| Row group optimization | Automatic | Manual | No |
| Validation | `gpio check` | No | No |
| Partitioning | Built-in | Manual | No |
| STAC generation | Built-in | No | No |

### DuckDB - For Advanced Operations

Use DuckDB when gpio doesn't support a specific operation (complex SQL, joins, aggregations, geometry operations). **Always apply best practices manually:**

```sql
-- Load spatial extension
LOAD spatial;

-- Convert with proper settings
COPY (
    SELECT *, ST_Envelope(geometry) as bbox
    FROM read_parquet('input.parquet')
    ORDER BY ST_Hilbert(geometry)  -- Critical: spatial sort
) TO 'output.parquet' (
    FORMAT PARQUET,
    COMPRESSION ZSTD,
    COMPRESSION_LEVEL 15,          -- Use 15+ for distribution
    ROW_GROUP_SIZE 100000
);
```

**DuckDB best practices checklist:**
- [ ] `ORDER BY ST_Hilbert(geometry)` - spatial sorting
- [ ] `COMPRESSION ZSTD` with `COMPRESSION_LEVEL 15`
- [ ] `ROW_GROUP_SIZE 100000` (50k-150k range)
- [ ] Include bbox column if needed: `ST_Envelope(geometry) as bbox`
- [ ] Run `gpio check all output.parquet` to validate

### When Each Tool is Best

| Task | Use |
|------|-----|
| Convert formats to GeoParquet | gpio |
| Validate GeoParquet files | gpio |
| Partition large datasets | gpio |
| Generate STAC metadata | gpio |
| Upload to cloud storage | gpio |
| Complex SQL/joins/aggregations | DuckDB |
| Geometry operations (buffer, union) | DuckDB |
| Reading for analysis | DuckDB or GeoPandas |

---

## gpio Command Reference

### Inspection & Analysis

```bash
# Quick overview
gpio inspect <file>

# Preview rows
gpio inspect head <file> --count 5
gpio inspect tail <file> --count 5

# Statistics
gpio inspect stats <file>

# Full metadata (GeoParquet spec, Parquet structure)
gpio inspect meta <file>
gpio inspect meta <file> --json
```

### Validation

```bash
# Run all checks
gpio check all <file>

# Individual checks
gpio check compression <file>
gpio check bbox <file>
gpio check spatial <file>
gpio check row-group <file>
gpio check spec <file>

# Auto-fix issues
gpio check all <file> --fix --output <fixed_file>
```

### Format Conversion

```bash
# Convert to GeoParquet (Hilbert sorted by default)
gpio convert geoparquet <input> <output>

# Skip Hilbert sorting (faster, less optimized)
gpio convert geoparquet <input> <output> --skip-hilbert

# High compression for distribution
gpio convert geoparquet <input> <output> --compression-level 15

# Convert GeoParquet to GeoJSON
gpio convert geojson <input> <output>

# Reproject
gpio convert reproject <input> <output> --target-crs EPSG:4326
```

### Data Extraction

```bash
# Extract by bounding box
gpio extract <input> <output> --bbox "minx,miny,maxx,maxy"

# Extract columns
gpio extract <input> <output> --columns "id,name,geometry"

# SQL WHERE filter
gpio extract <input> <output> --where "population > 10000"

# Limit rows
gpio extract <input> <output> --limit 1000
```

### Sorting

```bash
# Hilbert curve (best for spatial queries)
gpio sort hilbert <input> <output>

# Sort by column
gpio sort column <input> <output> --column "timestamp"

# Quadkey ordering
gpio sort quadkey <input> <output>
```

### Adding Columns

```bash
# Add bbox column
gpio add bbox <input> <output>

# Add covering metadata to existing bbox
gpio add bbox-metadata <file>

# Add spatial index columns
gpio add quadkey <input> <output> --resolution 12
gpio add h3 <input> <output> --resolution 9
```

### Partitioning

```bash
# KD-tree (balanced file sizes)
gpio partition kdtree <input> <output_dir> --max-rows-per-file 500000

# Admin boundaries
gpio partition admin <input> <output_dir>

# Quadkey grid
gpio partition quadkey <input> <output_dir> --resolution 6

# By column value
gpio partition string <input> <output_dir> --column "region"
```

### Publishing

```bash
# Generate STAC metadata
gpio publish stac <input> <output.json>
gpio publish stac <input_dir> <output_dir> --collection-id "dataset-name"

# Upload to S3
gpio publish upload <file> s3://bucket/path/file.parquet
gpio publish upload <dir> s3://bucket/path/ --recursive
```

### Remote Files

```bash
# gpio can read from cloud storage directly
gpio inspect s3://bucket/file.parquet
gpio inspect https://example.com/data.parquet

# Private S3 with profile
gpio inspect s3://bucket/file.parquet --profile my-profile
```

---

## Workflow: Creating Optimized GeoParquet

When a user provides spatial data:

### 1. Understand the Source
- What format? (GeoJSON, Shapefile, FlatGeobuf, GeoPackage, CSV, Parquet)
- Local file or URL?
- How large? (rows and MB)

### 2. Explore the Data
```bash
gpio inspect <file>
gpio inspect stats <file>
```
Report: row count, geometry type, CRS, columns, file size.

### 3. Convert with Best Practices
```bash
# Standard (fast)
gpio convert geoparquet <input> <output>

# For distribution (higher compression)
gpio convert geoparquet <input> <output> --compression-level 15
```

### 4. Validate
```bash
gpio check all <output>

# Fix issues if found
gpio check all <output> --fix --output <fixed>
```

### 5. Optimize Based on Size

**Small (<100MB, <100k rows):**
- Single file, Hilbert sorted, bbox column

**Medium (100MB-2GB, 100k-10M rows):**
- Single file, Hilbert sorted, bbox + covering metadata
- Row groups 100k, compression level 15

**Large (>2GB, >10M rows):**
- Partition (kdtree, admin, or quadkey)
- Generate STAC metadata
- Consider if full dataset is needed or extraction suffices

### 6. Publish
```bash
# Generate STAC
gpio publish stac <file> <file.stac.json>

# Upload
gpio publish upload <file> s3://bucket/path/
```

---

## Tips

- `--verbose` for detailed output
- `--dry-run` to preview operations
- `--json` for machine-readable output
- Always validate with `gpio check all` before publishing
- For very large files, warn user about processing time
