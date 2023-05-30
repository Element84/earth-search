# Earth Search STAC API

This README contains information on Element 84's [Earth Search STAC API](https://earth-search.aws.element84.com/v1).
Earth Search is a freely to use SpatioTemporal Asset Catalog (STAC) API containing an index of geospatial data collections
available on [AWS's Registry of Open Data](https://aws.amazon.com/earth/) (RODA).

This public API does not come with any guaranteed service. If you are using Earth Search in production
and are interested in a private data catalog that can incorporate both public and private data sources,
please reach out. Earth Search is powered by [FilmDrop](https://element84.com/filmdrop), a collection of
open-source applications and libraries.

To stay up to date with any new changes or datasets to the Earth Search API, please sign up for the
[mailing list]().

## General Notes

### Extensions Used

All the Earth Search Collections make use of the following extensions:

- [Grid](https://github.com/stac-extensions/grid) (for any tiled data)
- [Processing](https://github.com/stac-extensions/processing) for indicating software libraries and versions
- [Projection](https://github.com/stac-extensions/projection)
- [Raster](https://github.com/stac-extensions/raster)
- [View](https://github.com/stac-extensions/view)

In addition the [EO extension](https://github.com/stac-extensions/eo) is used for multispectral data and the
[SAR extension](https://github.com/stac-extensions/sar) is used for SAR data.

### s3 vs http URLs

When STAC Item assets (the data files) are in a requester pays bucket they are provided with an `s3://` style URL,
since AWS credentials are needed to access. For publicly available assets, an `https://` URL is used since no
authentication is needed. The Item/Assets also indicate if it's a requester pays bucket via the
[STAC storage extension](https://github.com/stac-extensions/storage)

## Collections

The STAC Collections in the API are detailed below along with specific notes on each dataset including
any known issues. Please file an issue in this repository for any new issues, such as incorrect metadata.
Some Collections do have missing Items due to several issues and these are being worked on.

| Collection | Public Dataset | stactools package | # of Items |
| -------------- | ---------- | ----------------- | ---------- |
| [`cop-dem-glo-30`](https://earth-search.aws.element84.com/v1/collections/cop-dem-glo-30) | [Copernicus DEM](https://registry.opendata.aws/copernicus-dem/) | [cop-dem](https://github.com/stactools-packages/cop-dem) | ~ 26,000 |
| [`cop-dem-glo-9`0](https://earth-search.aws.element84.com/v1/collections/cop-dem-glo-90) | [Copernicus DEM](https://registry.opendata.aws/copernicus-dem/) | [cop-dem](https://github.com/stactools-packages/cop-dem) | ~ 26,000 |
| [`landsat-c2-l2`](https://earth-search.aws.element84.com/v1/collections/landsat-c2-l2) | [USGS Landsat](https://registry.opendata.aws/usgs-landsat/) | [landsat](https://github.com/stactools-packages/landsat) | ~8.5 million |
| [`naip`](https://earth-search.aws.element84.com/v1/collections/naip) | [NAIP](https://registry.opendata.aws/naip/) | [naip](https://github.com/stactools-packages/naip) | ~1.2 million |
| [`sentinel-1-grd`](https://earth-search.aws.element84.com/v1/collections/sentinel-1-grd) | [Sentinel-1 GRD](https://registry.opendata.aws/sentinel-1/) | [sentinel1](https://github.com/stactools-packages/sentinel1) | ~2.7 million |
| [`sentinel-2-l1c`](https://earth-search.aws.element84.com/v1/collections/sentinel-2-l1c) | [Sentinel-2](https://registry.opendata.aws/sentinel-2/) |[sentinel2](https://github.com/stactools-packages/sentinel2) | ~21.7 million |
| [`sentinel-2-l2a`](https://earth-search.aws.element84.com/v1/collections/sentinel-2-l2a) | [Sentinel-2](https://registry.opendata.aws/sentinel-2/) and [Sentinel-2 COGS](https://registry.opendata.aws/sentinel-2-l2a-cogs/) | [sentinel2](https://github.com/stactools-packages/sentinel2) | ~21.7 million |

### Copernicus DEM

The Copernicus DEM data is available in two different Collections with the only difference being the 
resolution: 30 meter vs 90 meter.

### Landsat Collection 2

The `landsat-c2-l2` Collection consists of Items pointing to the authoritative source of Landsat data,
accessible through the official [USGS Landsat STAC API](https://landsatlook.usgs.gov/stac-server). The Earth
Search API combines two USGS Collections from that API: `landsat-c2l2-sr` (Surface Reflectrance for optical bands)
and `landsat-c2l2-st` (Surface Temperature for thermal bands) as a matter of convenience when using
both the optical and thermal bands.

Note that while the majority of Items contain both thermal and optical, some Items contain only optical with
no thermal bands. There are no Items with only thermal data. This is reflected in the Item property `instruments`,
an array indicating the sensors used. For example for Landsat-9 instruments will be either `['oli', 'tirs']`,
or just `['oli']`. Each Landsat Item also contains links back to the USGS Item(s).

The Landsat ID in Earth Search is different from the USGS ID in that it does not include a processing date,
thereby ensuring the ID does not change for a given scene.

#### Known Issues

- When USGS reprocesses an Item, the newly processed datafiles replace the old Items in Earth Search (because the ID
is the same). However there are some historical scenes in the index that do not point to the most recently processed
scene, which means the assets contain dead links.

### NAIP

NAIP data is released by year and by each individual state. Most states have a 3 year release cycle, although North Dakota
is released yearly. To find data for a specific naip year use the STAC Item property `naip:year` and for state, `naip:state`.

NAIP has historically been for the Continental United States only, however starting in 2021 Hawaii was included in the dataset.

### Sentinel-1

The Sentinel-1 data is the Ground-Range Detected amplitude-only data. Sentinel-1 is not distributed in tiles, instead
images can vary substantially in coverage and size. The original footprints of the scenes were found to not be very
accurate, so as part of the indexing process for Earth Search, new footprints are generated.

### Sentinel-2

The Sentinel-2 Collections represent the vast majority of Items stored in Earth Search, especially since both the Level-1 and
Level-2 are included. The Level-1 data is the original JP2K files, and the Level-2 data includes a set of assets for the
original JP2K files, as well as a set of assets for the Cloud-Optimized GeoTIFF (COG) versions.

#### Gain/Offset in Items after Jan 25, 2022

[Starting Jan 25, 2022](https://sentinels.copernicus.eu/web/sentinel/-/copernicus-sentinel-2-processing-baseline-04-00-25-01-2022),
the Copernicus program started processing using a new version of software, called the Baseline 04.00. Part of the
changes included an offset that needs to be applied to the data when read in order to compare to any pre-04.00 processed data.

As part of the conversion to COGs, the offset has been applied to some of the Items. To determine if the offset has been applied
to the COGs or not, examine the asset of interest. The `raster:bands` field has fields for both `scale` and `offset`.  If the
`offset` is anything other than 0, then it should be applied after the scale factor to correct the data.

#### Known Issues

- Most of the scenes crossing the antimeridian are excluded from the Catalog. This issue has been resolved in the
stactools-sentinel2 package by splitting polygons crossing the antimeridian. The historical scenes will be indexed
in the coming months.
- There are some missing Items from the Catalog because currently the L1 and L2 data are ingested together and in some cases
there is no available L1 data which causes the ingest to fail. The causes for this are being explored and we will likely move
to ingesting the L2 data regardless if L1 is available or not.

## Contributions

The Earth Search API is made available as a best effort by Element 84. While it is not possible for the public to
aid in the operation of the pipeline, the creation of the STAC metadata uses repos from the open-source
[stactools-packages](https://github.com/stactools-packages) organization. PRs are welcome to these repositories which
may eventually make their way into Earth Search depending on their backwards compatability.

The best way to contribute is to provide feedback via issues in this repository.

## Updates

May 31, 2023
- Earth Search STAC API v1 released