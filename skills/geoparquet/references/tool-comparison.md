# GeoParquet Tool Comparison

## Feature Matrix

| Feature | gpio | GDAL/ogr2ogr (3.9+) | GeoPandas |
|---------|------|---------------------|-----------|
| GeoParquet 1.1 spec | Full | Full | Partial |
| Spatial sorting | Hilbert (fast) | SORT_BY_BBOX (slow*) | Hilbert (manual) |
| bbox + covering metadata | Automatic | WRITE_COVERING_BBOX option | Manual |
| zstd compression | Default | Supported (option) | Supported (option) |
| Row group optimization | Automatic | ROW_GROUP_SIZE option | Supported (option) |
| Validation | `gpio check` | No | No |
| Partitioning | Built-in | No | Manual |
| STAC generation | Built-in | No | No |
| BigQuery/ArcGIS extraction | Built-in | No | No |
| Best practices by default | Yes | Requires explicit options | Requires explicit options |

*GDAL's SORT_BY_BBOX requires temporary conversion to GeoPackage, making it slower than gpio's native Hilbert sort.

## When to Use Each Tool

| Task | Recommended Tool |
|------|------------------|
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
| Convert obscure formats | GDAL (broader format support) |

## Key Differences

### gpio vs GDAL

**Speed**: gpio's Hilbert sorting is native and fast; GDAL's SORT_BY_BBOX converts to GeoPackage first, adding overhead.

**Defaults**: gpio applies best practices automatically. GDAL requires explicit options:
```bash
ogr2ogr -f Parquet output.parquet input.geojson \
    -lco SORT_BY_BBOX=YES \
    -lco WRITE_COVERING_BBOX=YES \
    -lco COMPRESSION=ZSTD
```

**When to use GDAL**: When gpio doesn't support the input format. GDAL has broader format support.

### gpio vs GeoPandas

**Defaults**: gpio applies best practices automatically. GeoPandas requires explicit options:
```python
import geopandas as gpd

gdf = gpd.read_file("input.geojson")
# Must manually sort, set compression, etc.
gdf.to_parquet("output.parquet", compression="zstd")
```

**When to use GeoPandas**: For analysis workflows in notebooks. Just set compression/sorting options when writing final output for distribution.

### gpio vs DuckDB

**Best practices**: gpio handles GeoParquet best practices automatically. DuckDB requires manual configuration:

```sql
LOAD spatial;

COPY (
    SELECT *, ST_Envelope(geometry) as bbox
    FROM read_parquet('input.parquet')
    ORDER BY ST_Hilbert(geometry)
) TO 'output.parquet' (
    FORMAT PARQUET,
    COMPRESSION ZSTD,
    COMPRESSION_LEVEL 15,
    ROW_GROUP_SIZE 100000
);
```

**When to use DuckDB**: Complex SQL, joins, aggregations, geometry operations (buffer, union, intersection). Requires DuckDB 1.5+ for projection/CRS operations.

**DuckDB best practices checklist:**
- [ ] `ORDER BY ST_Hilbert(geometry)` - spatial sorting
- [ ] `COMPRESSION ZSTD` with `COMPRESSION_LEVEL 15`
- [ ] `ROW_GROUP_SIZE 100000` (50k-150k range)
- [ ] Include bbox column: `ST_Envelope(geometry) as bbox`
- [ ] Validate output: `gpio check all output.parquet`

## Installation

### gpio
```bash
# Isolated install (recommended)
pipx install --pre geoparquet-io

# Or with pip/uv
pip install --pre geoparquet-io
uv pip install --pre geoparquet-io
```

Note: Use `--pre` for latest beta releases (1.0 not yet released).

### DuckDB
```bash
pip install "duckdb>=1.5"
```

Requires 1.5+ for projection/CRS operations.

### gpio-pmtiles plugin
```bash
# If gpio installed via pipx
pipx inject geoparquet-io gpio-pmtiles

# Or with pip
pip install gpio-pmtiles
```
