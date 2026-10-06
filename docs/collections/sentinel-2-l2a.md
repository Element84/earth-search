# Sentinel-2 L2A (sentinel-2-l2a and sentinel-2-c1-l2a)

- Collections:
  - v2 (beta): [`sentinel-2-l2a`](https://earth-search.aws.element84.com/v2/collections/sentinel-2-l2a)
  - v1: [`sentinel-2-l2a`](https://earth-search.aws.element84.com/v1/collections/sentinel-2-l2a)
  - v1: [`sentinel-2-c1-l2a`](https://earth-search.aws.element84.com/v1/collections/sentinel-2-c1-l2a)
- Public dataset: [Sentinel-2](https://registry.opendata.aws/sentinel-2/) and [Sentinel-2 COGS](https://registry.opendata.aws/sentinel-2-l2a-cogs/) (`sentinel-2-c1-l2a`: tbd)
- stactools package: [sentinel2](https://github.com/stactools-packages/sentinel2)
- Number of Items: ~26.2 million (v2 `sentinel-2-l2a`), ~51.6 million (v1 `sentinel-2-l2a`), ~30.6 million (v1 `sentinel-2-c1-l2a`)

The Sentinel-2 Collections represent the vast majority of Items stored in Earth Search, especially since both the Level-1 and
Level-2 are included. The Level-2 data includes a set of assets for the original JPEG 2000 files, as well as a set of
assets for the Cloud-Optimized GeoTIFF (COG) versions. See also [`sentinel-2-l1c`](sentinel-2-l1c.md).

## v1 vs v2

> [!NOTE]
> Earth Search v2 is currently in beta and may change before general release.

Earth Search v1 has two Sentinel-2 L2A collections: `sentinel-2-l2a` and `sentinel-2-c1-l2a` (Collection 1).
In Earth Search v2, the Collection 1 data is provided in `sentinel-2-l2a`, and `sentinel-2-c1-l2a` no longer exists.
The Item ID is unchanged between `sentinel-2-c1-l2a` in v1 and `sentinel-2-l2a` in v2.

| Earth Search | Collection          | Content                                                              |
| ------------ | ------------------- | -------------------------------------------------------------------- |
| v1           | `sentinel-2-l2a`    | Original JPEG 2000 and COG assets, all processing baselines          |
| v1           | `sentinel-2-c1-l2a` | COGs processed to at least baseline 5.0 (Collection 1)               |
| v2 (beta)    | `sentinel-2-l2a`    | Replaces v1 `sentinel-2-c1-l2a`                                      |

### Changes from v1 `sentinel-2-c1-l2a` to v2 `sentinel-2-l2a`

- Collection changed from `sentinel-2-c1-l2a` to `sentinel-2-l2a`. The Item ID is unchanged.
- STAC version advanced from 1.0.0 to 1.1.0, with updated EO, projection, raster, and storage extension versions.
  - `eo`: 1.1.0 → 2.0.0
  - `projection`: 1.1.0 → 2.0.0
  - `raster`: 1.1.0 → 2.0.0
  - `storage`: 1.0.0 → 2.0.0
- Only indexes collection 1 processing versions (`05.00`+, excluding `05.09`)
- Storage metadata uses storage schemes and refs
- Multi-bands items use new 1.1.0 bands construct
- The `tileinfo_metadata` asset and the v1 `canonical`, `via`, and `thumbnail` links are removed.

## Collection 1 (sentinel-2-c1-l2a)

The Sentinel-2 Collection 1 L2A (`sentinel-2-c1-l2a`) was intended to eventually
replace the other v1 Sentinel-2 L2A collection (`sentinel-2-l2a`). Items in this collection
only contain assets referring to COGs of data processed to at least baseline 5.0. 

Version 2 of the STAC API excludes baseline 5.09, as it was removed from collection 1.

### S3 Data

As part of Earth Search, Cloud-optimized GeoTIFFs (COGs) are generated for the
Sentinel-2 Collection 1 Level-2A dataset from the ESA/Sinergise JPEG 2000 files.
These are referenced in the
Earth Search API Collection `sentinel-2-c1-l2a`. This data is stored in the
S3 Bucket `e84-earth-search-sentinel-data`.

#### Inventory Report

[S3 inventory reports](https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage-inventory.html)
are generated for the `e84-earth-search-sentinel-data` bucket and stored in the
`e84-earth-search-sentinel-data-inventory` bucket. Inventory reports are generated daily
and stored as parquet files, under the `primary` configuration name,
e.g. `s3://e84-earth-search-sentinel-data-inventory/e84-earth-search-sentinel-data/primary/`

## Gain/Offset in Items after Jan 25, 2022

[Starting Jan 25, 2022](https://sentinels.copernicus.eu/web/sentinel/-/copernicus-sentinel-2-processing-baseline-04-00-25-01-2022),
the Copernicus program started processing using a new version of software, called the Baseline 04.00. Part of the
changes included an offset that needs to be applied to the data when read in order to compare to any pre-04.00 processed data.

As part of the conversion to COGs, the offset has been applied to some of the Items. To determine if the offset has been applied
to the COGs or not, examine the asset of interest. The `raster:bands` field has fields for both `scale` and `offset`.  If the
`offset` is anything other than 0, then it should be applied after the scale factor to correct the data.

## Known Issues

- Most of the scenes crossing the antimeridian are excluded from the Catalog. This issue has been resolved in the
stactools-sentinel2 package by splitting polygons crossing the antimeridian. The historical scenes will be indexed
in the coming months.
- There are some missing Items from the Catalog because currently the L1 and L2 data are ingested together and in some cases
there is no available L1 data which causes the ingest to fail. The causes for this are being explored and we will likely move
to ingesting the L2 data regardless if L1 is available or not.

Back to the [Earth Search README](../../README.md#collections).
