# gpio Command Reference

Complete reference for all gpio CLI commands.

## Inspection & Analysis

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

## Validation

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

## Format Conversion

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

## Data Extraction & Subsetting

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

## Extract from BigQuery

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

## Extract from ArcGIS Feature Services

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

## Sorting

```bash
# Hilbert curve (best for spatial queries)
gpio sort hilbert <input> <output>

# Sort by column
gpio sort column <input> <output> --column "timestamp"

# Quadkey ordering
gpio sort quadkey <input> <output>
```

## Adding Columns & Spatial Indices

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

## Partitioning

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

## Publishing

```bash
# Generate STAC metadata
gpio publish stac <input> <output.json>
gpio publish stac <input_dir> <output_dir> --collection-id "dataset-name"

# Upload to S3
gpio publish upload <file> s3://bucket/path/file.parquet
gpio publish upload <dir> s3://bucket/path/ --recursive
```

## Remote Files

```bash
# gpio can read from cloud storage directly
gpio inspect s3://bucket/file.parquet
gpio inspect https://example.com/data.parquet

# Private S3 with profile
gpio inspect s3://bucket/file.parquet --profile my-profile
```

## Convert to PMTiles (Vector Tiles)

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

## Common Options

Most commands accept these options:

```bash
--verbose           # Detailed output
--dry-run           # Preview operations without executing
--json              # Machine-readable output
--overwrite         # Overwrite existing files
--compression       # zstd|snappy|gzip|lz4|brotli|none (default: zstd)
--compression-level # 1-22 for zstd (default: 15 for most commands)
--row-group-size    # Rows per group (default: 100000)
--aws-profile       # AWS profile for S3 operations
```
