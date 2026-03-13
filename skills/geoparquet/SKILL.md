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
pipx install --pre geoparquet-io

# Or with pip/uv
pip install --pre geoparquet-io
uv pip install --pre geoparquet-io

# Verify
gpio --version
```

Note: Use `--pre` to get the latest beta releases (1.0 is not yet released).

If gpio is not installed, help the user install it before proceeding.

**Why gpio over alternatives:**

| Feature | gpio | GDAL/ogr2ogr (3.9+) | GeoPandas |
|---------|------|---------------------|-----------|
| GeoParquet 1.1 spec | Full | Full | Partial |
| Spatial sorting | Hilbert (optimal) | SORT_BY_BBOX | No |
| bbox + covering metadata | Automatic | WRITE_COVERING_BBOX option | Manual |
| zstd compression | Default | Supported (option) | No |
| Row group optimization | Automatic | ROW_GROUP_SIZE option | Manual |
| Validation | `gpio check` | No | No |
| Partitioning | Built-in | No | Manual |
| STAC generation | Built-in | No | No |
| BigQuery/ArcGIS extraction | Built-in | No | No |
| Best practices by default | Yes | Requires explicit options | No |

**Key differences:**
- **Hilbert vs SORT_BY_BBOX**: Hilbert curves provide optimal spatial clustering; SORT_BY_BBOX is simpler but less effective
- **Defaults**: gpio applies best practices automatically; GDAL requires explicit options like `-lco SORT_BY_BBOX=YES -lco WRITE_COVERING_BBOX=YES -lco COMPRESSION=ZSTD`
- **GDAL is excellent** for format conversion and has broad format support; use it when gpio doesn't support your input format

### DuckDB - For Advanced Operations

Use DuckDB when gpio doesn't support a specific operation (complex SQL, joins, aggregations, geometry operations). **Requires DuckDB 1.5+ for projection/CRS operations.**

```bash
# Install DuckDB 1.5+
pip install "duckdb>=1.5"
```

**Always apply best practices manually:**

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
| Extract subsets from large remote files | gpio |
| Extract from BigQuery | gpio |
| Extract from ArcGIS Feature Services | gpio |
| Add spatial indices (H3, S2, quadkey) | gpio |
| Add administrative boundaries | gpio |
| Validate GeoParquet files | gpio |
| Partition large datasets | gpio |
| Generate STAC metadata | gpio |
| Upload to cloud storage | gpio |
| Convert to PMTiles | gpio-pmtiles or tippecanoe |
| Complex SQL/joins/aggregations | DuckDB |
| Geometry operations (buffer, union, reproject) | DuckDB (1.5+) |
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

### Data Extraction & Subsetting

Efficiently extract subsets from GeoParquet files, including large remote files (S3, HTTPS). Uses bbox metadata for fast spatial filtering.

```bash
# Extract by bounding box (uses bbox column for fast filtering)
gpio extract data.parquet output.parquet --bbox "-122.5,37.5,-122.0,38.0"

# Extract by geometry (GeoJSON, WKT, or file)
gpio extract data.parquet output.parquet --geometry @boundary.geojson
gpio extract data.parquet output.parquet --geometry "POLYGON((...)))"

# Select specific columns
gpio extract data.parquet output.parquet --include-cols "id,name,area"

# Exclude columns
gpio extract data.parquet output.parquet --exclude-cols "internal_id,temp"

# SQL WHERE filter
gpio extract data.parquet output.parquet --where "population > 10000"

# Limit rows
gpio extract data.parquet output.parquet --limit 1000

# Combine filters
gpio extract data.parquet output.parquet \
    --bbox "-122.5,37.5,-122.0,38.0" \
    --include-cols "id,name" \
    --where "status = 'active'" \
    --limit 5000

# Merge multiple files with glob pattern
gpio extract "data/*.parquet" merged.parquet

# Extract from remote files (efficient - only downloads needed data)
gpio extract s3://bucket/large-file.parquet local.parquet \
    --bbox "-122.5,37.5,-122.0,38.0" \
    --aws-profile my-profile

gpio extract https://data.source.coop/file.parquet local.parquet \
    --bbox "-122.5,37.5,-122.0,38.0"
```

### Extract from BigQuery

Extract spatial data from BigQuery tables to optimized GeoParquet. GEOGRAPHY columns are automatically converted.

```bash
# Extract entire table
gpio extract bigquery myproject.geodata.buildings output.parquet

# With filtering (pushed to BigQuery for efficiency)
gpio extract bigquery myproject.geodata.buildings output.parquet \
    --where "area > 1000" \
    --bbox "-122.5,37.5,-122.0,38.0" \
    --limit 10000

# Select specific columns
gpio extract bigquery myproject.geodata.buildings output.parquet \
    --include-cols "id,name,geography"

# Using service account
gpio extract bigquery myproject.geodata.buildings output.parquet \
    --credentials-file /path/to/service-account.json
```

**Authentication:** Uses gcloud auth, GOOGLE_APPLICATION_CREDENTIALS, or --credentials-file.

### Extract from ArcGIS Feature Services

Download features from ArcGIS REST services to optimized GeoParquet.

```bash
# Public service (no auth)
gpio extract arcgis https://services.arcgis.com/.../FeatureServer/0 output.parquet

# With server-side filtering (efficient - reduces download)
gpio extract arcgis https://services.arcgis.com/.../FeatureServer/0 output.parquet \
    --bbox "-122.5,37.5,-122.0,38.0" \
    --where "state='CA'" \
    --include-cols "name,population" \
    --limit 1000

# With authentication
gpio extract arcgis https://services.arcgis.com/.../FeatureServer/0 output.parquet \
    --token "your-token"

gpio extract arcgis https://services.arcgis.com/.../FeatureServer/0 output.parquet \
    --username user --password pass
```

**Note:** Filters are pushed to the server for efficiency. Output includes Hilbert sorting and bbox metadata by default.

### Sorting

```bash
# Hilbert curve (best for spatial queries)
gpio sort hilbert <input> <output>

# Sort by column
gpio sort column <input> <output> --column "timestamp"

# Quadkey ordering
gpio sort quadkey <input> <output>
```

### Adding Columns & Spatial Indices

Enrich GeoParquet with spatial indices, administrative boundaries, and bbox metadata.

```bash
# Bbox column + covering metadata
gpio add bbox <input> <output>
gpio add bbox-metadata <file>  # Add metadata to existing bbox

# H3 hexagonal cells (resolution 0-15, default 9)
# Res 7: ~5km², Res 9: ~105m², Res 11: ~2m²
gpio add h3 <input> <output> --resolution 9

# S2 spherical cells (level 0-30, default 13)
# Level 8: ~1,250km², Level 13: ~1.2km², Level 18: ~1,200m²
gpio add s2 <input> <output> --level 13

# Quadkey (Bing Maps tiles)
gpio add quadkey <input> <output> --resolution 12

# A5 cells
gpio add a5 <input> <output>

# KD-tree cell IDs (for balanced partitioning)
gpio add kdtree <input> <output>

# Administrative divisions via spatial join
# GAUL dataset: adds gaul_continent, gaul_country, gaul_department
gpio add admin-divisions <input> <output> --dataset gaul

# Overture Maps: adds overture_country, overture_region
gpio add admin-divisions <input> <output> --dataset overture

# Select specific levels
gpio add admin-divisions <input> <output> --dataset gaul --levels "continent,country"

# Custom column prefix
gpio add admin-divisions <input> <output> --dataset gaul --prefix "admin"
```

**Admin divisions notes:**
- Datasets are cached locally after first download (~5-50MB)
- Input data must be in WGS84 or compatible CRS
- Use `--clear-cache` to refresh cached datasets

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

### Convert to PMTiles (Vector Tiles)

Generate PMTiles for web map display using the gpio-pmtiles plugin.

**Installation:**
```bash
# If gpio installed via pipx
pipx inject geoparquet-io gpio-pmtiles

# Or with pip
pip install gpio-pmtiles
```

**Usage:**
```bash
# Basic conversion
gpio pmtiles create buildings.parquet buildings.pmtiles

# With filtering (applied before tile generation)
gpio pmtiles create data.parquet tiles.pmtiles \
    --bbox "-122.5,37.5,-122.0,38.0" \
    --where "population > 10000"
```

**Alternative: Pipe through tippecanoe:**
```bash
# Stream GeoJSON to tippecanoe (no intermediate files)
gpio convert geojson buildings.parquet | tippecanoe -P -o buildings.pmtiles

# With filtering and reduced precision for smaller output
gpio extract large.parquet --bbox "-122.5,37.5,-122,38" | \
    gpio convert geojson - --precision 5 | \
    tippecanoe -P -o output.pmtiles
```

**Tips:**
- Filter data first with `gpio extract` to reduce processing time
- Use `--precision 5` or `6` for smaller GeoJSON output
- The `-P` flag enables tippecanoe's parallel processing mode

---

## Workflow: Creating Optimized GeoParquet

When a user provides spatial data:

### 1. Understand the Source
- What format? (GeoJSON, Shapefile, FlatGeobuf, GeoPackage, CSV, Parquet)
- Local file, URL, cloud storage (S3/GCS/Azure)?
- External service? (BigQuery table, ArcGIS Feature Service)
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
