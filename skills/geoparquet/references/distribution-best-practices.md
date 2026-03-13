# GeoParquet Distribution Best Practices

Based on the [OGC GeoParquet distribution guidelines](https://github.com/opengeospatial/geoparquet/blob/main/format-specs/distributing-geoparquet.md).

## Compression

**Use zstd compression at level 15** for distribution.

- Higher levels (17+) require significantly more computation for <1% additional compression
- zstd achieves better compression ratios than snappy with similar decompression speeds
- Decompression time is consistent across compression levels

```bash
# gpio uses zstd by default (level 3 for speed during development)
# For distribution, use higher compression:
gpio convert geoparquet input.geojson output.parquet --compression-level 15
```

## Spatial Ordering (Critical)

**Data must be spatially sorted** for bbox filtering to work efficiently.

Without spatial ordering, queries must scan the entire file even with bbox metadata. Spatial sorting clusters nearby features together in the same row groups.

Options for spatial sorting:
- **Hilbert curve** (best): Optimal spatial clustering, preserves locality
- **Quadkey/S2/GeoHash**: Grid-based alternatives
- **SORT_BY_BBOX**: GDAL's simpler approach (requires temp GeoPackage)

```bash
# gpio applies Hilbert sorting by default
gpio convert geoparquet input.geojson output.parquet

# Explicit Hilbert sort on existing file
gpio sort hilbert input.parquet output.parquet
```

## Bbox Column + Covering Metadata

**Include a bbox column** with GeoParquet 1.1 covering metadata.

This struct of four floats (xmin, ymin, xmax, ymax) enables row-group-level spatial filtering. The covering metadata tells readers which column contains the bounding boxes.

```bash
# Add bbox column
gpio add bbox input.parquet output.parquet

# Add covering metadata to existing bbox column
gpio add bbox-metadata file.parquet
```

## Row Group Size

**Target 50,000-150,000 rows per group** (or 128-256MB).

This balances:
- **Smaller groups**: Better spatial query latency, faster bbox filtering
- **Larger groups**: Better full-scan analytics performance

```bash
gpio convert geoparquet input.geojson output.parquet --row-group-size 100000
```

## Partitioning Large Datasets

**Partition datasets >2GB** into multiple spatially-organized files.

Partitioning strategies:
- **KD-tree**: Balanced splits by feature count (recommended)
- **Admin boundaries**: Country/region based (good for political data)
- **S2/Quadkey/GeoHash**: Grid-based (uniform cells)

```bash
# KD-tree partitioning (recommended for balanced files)
gpio partition kdtree input.parquet output_dir/ --max-rows-per-file 500000

# Admin boundary partitioning
gpio partition admin input.parquet output_dir/

# Quadkey grid partitioning
gpio partition quadkey input.parquet output_dir/ --resolution 6
```

## STAC Metadata

Use STAC (SpatioTemporal Asset Catalog) for discovery.

- Media type: `application/vnd.apache.parquet`
- For partitioned data, create a Collection with Items for each file
- Each Item should include the file's bounding box

```bash
# Single file
gpio publish stac data.parquet data.stac.json

# Partitioned directory (creates Collection + Items)
gpio publish stac ./partitioned/ ./partitioned/ --collection-id "my-dataset"
```

## Summary Checklist

For distribution-ready GeoParquet:

- [ ] zstd compression at level 15
- [ ] Hilbert spatial sorting applied
- [ ] bbox column with covering metadata
- [ ] Row groups 50k-150k rows
- [ ] Partitioned if >2GB
- [ ] STAC metadata generated
- [ ] Validated with `gpio check all`
