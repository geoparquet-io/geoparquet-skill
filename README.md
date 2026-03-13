# GeoParquet Skill

A skill for AI agents to work with GeoParquet files, powered by [geoparquet-io](https://github.com/geoparquet-io/geoparquet-io) (`gpio` CLI).

## What it does

This skill guides AI agents through the complete GeoParquet workflow:

- **Ingest** spatial data from any source (GeoJSON, Shapefile, FlatGeobuf, etc.)
- **Explore** data structure, schema, and statistics
- **Convert** to optimized GeoParquet format with Hilbert sorting
- **Validate** against GeoParquet spec and best practices
- **Partition** large datasets for efficient querying
- **Publish** to cloud storage with STAC metadata

## Installation

### Claude Code

```bash
# Add the marketplace
/plugin marketplace add geoparquet-io/geoparquet-skill

# Install the plugin
/plugin install geoparquet@geoparquet-skill
```

### Claude Cowork

**Via UI:**
1. Open Cowork → Click **Plugins** in the sidebar
2. Click **"+"** to add a new marketplace
3. Enter: `geoparquet-io/geoparquet-skill`
4. Select the **geoparquet** plugin and click **Add plugin**

**Via CLI:**
```bash
claude plugin marketplace add geoparquet-io/geoparquet-skill
claude plugin install geoparquet@geoparquet-skill
```

### OpenSkills (works with Cursor, Codex, Aider, etc.)

```bash
npx openskills install geoparquet-io/geoparquet-skill
```

### Manual Installation

Copy the `skills/geoparquet/` directory to `~/.claude/skills/geoparquet/`.

## Prerequisites

Install the `gpio` CLI tool:

```bash
# Isolated install (recommended)
pipx install --pre geoparquet-io

# Or with pip/uv
pip install --pre geoparquet-io
uv pip install --pre geoparquet-io
```

Note: Use `--pre` to get the latest beta releases (1.0 is not yet released).

Verify installation:

```bash
gpio --version
```

## Usage

Once installed, Claude will automatically use this skill when you work with spatial data. Example prompts:

- "Convert this GeoJSON to GeoParquet: https://example.com/data.geojson"
- "Inspect this parquet file and tell me about its spatial properties"
- "Partition this large GeoParquet file by country"
- "Check if this file follows GeoParquet best practices"

## Releasing New Versions

1. Update the `version` field in:
   - `.claude-plugin/plugin.json`
   - `.claude-plugin/marketplace.json`

2. Commit and push:
   ```bash
   git add .
   git commit -m "Release v1.x.x"
   git tag v1.x.x
   git push origin main --tags
   ```

3. Users update with:
   ```bash
   /plugin marketplace update
   ```

## License

Apache 2.0
