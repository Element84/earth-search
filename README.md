# Earth Search STAC API <!-- omit from toc -->

- [General Notes](#general-notes)
  - [Extensions Used](#extensions-used)
  - [s3 vs http URLs](#s3-vs-http-urls)
  - [SNS Notifications of Items](#sns-notifications-of-items)
  - [Sentinel-2 Collection 1 Level-2A S3 Data](#sentinel-2-collection-1-level-2a-s3-data)
- [Collections](#collections)
  - [Copernicus DEM (cop-dem-glo-30 and cop-dem-glo-90)](#copernicus-dem-cop-dem-glo-30-and-cop-dem-glo-90)
  - [Landsat Collection 2 Level-2 (landsat-c2-l2)](#landsat-collection-2-level-2-landsat-c2-l2)
    - [Known Issues](#known-issues)
  - [NAIP (naip)](#naip-naip)
  - [Sentinel-1 (sentinel-1-grd)](#sentinel-1-sentinel-1-grd)
  - [Sentinel-2 L1C and L2A (sentinel-2-l1c and sentinel-2-l2a)](#sentinel-2-l1c-and-l2a-sentinel-2-l1c-and-sentinel-2-l2a)
    - [Gain/Offset in Items after Jan 25, 2022](#gainoffset-in-items-after-jan-25-2022)
    - [Known Issues](#known-issues-1)
  - [Sentinel-2 Collection 1 L2A (sentinel-2-c1-l2a)](#sentinel-2-collection-1-l2a-sentinel-2-c1-l2a)
- [Contributions](#contributions)
- [Updates](#updates)
  - [May 31, 2023](#may-31-2023)

This README contains information on [Element 84](https://element84.com)'s [Earth Search STAC API](https://earth-search.aws.element84.com/v1).
Earth Search is a free-to-use SpatioTemporal Asset Catalog (STAC) API containing an index of geospatial data collections
available on the [AWS Registry of Open Data](https://aws.amazon.com/earth/) (RODA).

This public API does not come with any guaranteed service. If you are using Earth Search in production
and are interested in a private data catalog that can incorporate both public and private data sources,
[please reach out](https://www.element84.com/work-with-us). Earth Search is powered by [FilmDrop](https://element84.com/filmdrop), a collection of
open-source applications and libraries for geospatial data ingest and cataloging supported by [Element 84](https://element84.com).

To stay up to date with any new changes or datasets to the Earth Search API, please sign up for the [mailing list](http://eepurl.com/irrwDo), you can
also review [the mailing list archive](https://us13.campaign-archive.com/home/?u=a7a7fcb1ce46c4d001fc76289&id=38266ef009).
For a detailed writeup about the Earth Search v1 and the changes from v0, see [this blog post](https://www.element84.com/geospatial/introducing-earth-search-v1-new-datasets-now-available/).

## General Notes

### Extensions Used

All the Earth Search STAC Items make use of the following extensions:

- [Grid](https://github.com/stac-extensions/grid) (for most gridded datasets, except Landsat)
- [Processing](https://github.com/stac-extensions/processing) for indicating software libraries and versions
- [Projection](https://github.com/stac-extensions/projection)
- [Raster](https://github.com/stac-extensions/raster)
- [View](https://github.com/stac-extensions/view)

Additionally, the [EO extension](https://github.com/stac-extensions/eo) is used for multispectral data and the
[SAR extension](https://github.com/stac-extensions/sar) is used for SAR data.

### s3 vs http URLs

When STAC Item assets (the data files) are in a requester pays bucket they are provided with an `s3://` style URL,
since AWS credentials are needed to access. For publicly-available assets, an `https://` URL is used since no
authentication is needed. The Item/Assets also indicate if it's a requester pays bucket via the
[STAC storage extension](https://github.com/stac-extensions/storage)

### SNS Notifications of Items

Data and metadata provided by Earth Search are processed using the
[Cirrus](https://github.com/cirrus-geo/cirrus-geo) geospatial processing
platform, also maintained by Element 84.

Cirrus supports an SNS Topic to which all ingested STAC Items are published. For
Earth Search, the URN for this topic is `arn:aws:sns:us-west-2:608149789419:cirrus-es-prod-publish`.

Messages published to this topic can be filtered based on the following
attributes:

- collection (String): The STAC Collection of the STAC Item.
- datetime (String): The STAC Item `datetime` value, a string in RFC 3339 format.
- start_datetime (String): The STAC Item `start_datetime` value, a string in RFC 3339 format, if that field exists; otherwise, the `datetime` value.
- end_datetime (String): The STAC Item `end_datetime` value, a string in RFC 3339 format, if that field exists; otherwise, the `datetime` value.
- bbox.ll_lon (Number): The lower left (southwest) longitude of the Item bounding box
- bbox.ll_lat (Number): The lower left (southwest) latitude of the Item bounding box
- bbox.ur_lon (Number): The upper right (northeast) longitude of the Item bounding box
- bbox.ur_lat (Number): The upper right (northeast) latitude of the Item bounding box
- cloud_cover (Number): The STAC Item `eo:cloud_cover` value, a floating point value between 0 and 100.
- status (String): A value of either `created` or `updated`, as to whether the Item was newly-ingested (`created`) or updated an existing Item (`updated`).

Note that for the bounding box, the abbreviations lower left (ll) and
upper right (ur) are used to describe the bounding box, even though it
is more accurate to describe them using cardinal directions. This is
sometimes confusing, as in the case with an antimerdian-crossing scene
where the "lower left" point has a larger longitude coordinate than the
"upper right" point.

The SNS notification "Message" field is the STAC Item that was ingested.

The following CloudFormation template is an example of an SQS Queue with a subscription to
the Earth Search SNS topic.

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: A sample template

Parameters:
  EarthSearchPublishTopicArn:
    Description: Earth Search Publish SNS Topic ARN
    Type: String
    Default: arn:aws:sns:us-west-2:608149789419:cirrus-es-prod-publish

  EarthSearchPublishTopicRegion:
    Description: Earth Search Publish SNS Topic Region
    Type: String
    Default: us-west-2

Resources:
  IngestEarthSearch:
    Type: AWS::SQS::Queue
    Properties:
      RedrivePolicy:
        deadLetterTargetArn: !GetAtt IngestEarthSearchDLQ.Arn
        maxReceiveCount: 5
      VisibilityTimeout: 3600 # seconds, 6x function timeout

  IngestEarthSearchQueuePolicy:
    Type: AWS::SQS::QueuePolicy
    Properties:
      Queues:
        - !Ref IngestEarthSearch
      PolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Action: sqs:SendMessage
            Resource: !GetAtt IngestEarthSearch.Arn
            Principal:
              Service: sns.amazonaws.com
            Condition:
              ArnEquals:
                aws:SourceArn: !Ref EarthSearchPublishTopicArn

  IngestEarthSearchDLQ:
    Type: AWS::SQS::Queue

  IngestEarthSearchSnsSubscription:
    Type: AWS::SNS::Subscription
    Properties:
      Protocol: sqs
      Endpoint: !GetAtt IngestEarthSearch.Arn
      Region: !Ref EarthSearchPublishTopicRegion
      TopicArn: !Ref EarthSearchPublishTopicArn
      FilterPolicyScope: MessageAttributes
      FilterPolicy:
        collection:
          - sentinel-2-c1-l2a
          - landsat-c2-l2
        bbox.ll_lon:
          - numeric:
              - ">="
              - 51.9
        bbox.ll_lat:
          - numeric:
              - ">="
              - 16.6
        bbox.ur_lon:
          - numeric:
              - "<="
              - 59.9
        bbox.ur_lat:
          - numeric:
              - "<="
              - 26.4
```

### Sentinel-2 Collection 1 Level-2A S3 Data

As part of Earth Search, Cloud-optimized GeoTIFFs (COGs) are generated for the
Sentinel-2 Collection 1 Level-2A dataset from the ESA/Singergize JPEG 2000 files.
These are referenced in the
Earth Search API Collection `sentinel-2-c1-l2a`. This data is stored in the
S3 Bucket `e84-earth-search-sentinel-data` and an S3 Inventory is generated into
S3 Bucket `e84-earth-search-sentinel-data-inventory`.

## Collections

The STAC Collections in the API are detailed below along with specific notes on each dataset including
any known issues. Please file an issue in this repository for any new issues, such as incorrect metadata.
Some Collections do have missing Items due to several issues and these are being worked on.

| Collection                                                                                     | Public Dataset                                                                                                                    | stactools package                                            | # of Items    |
| ---------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------ | ------------- |
| [`cop-dem-glo-30`](https://earth-search.aws.element84.com/v1/collections/cop-dem-glo-30)       | [Copernicus DEM](https://registry.opendata.aws/copernicus-dem/)                                                                   | [cop-dem](https://github.com/stactools-packages/cop-dem)     | ~ 26,000      |
| [`cop-dem-glo-90`](https://earth-search.aws.element84.com/v1/collections/cop-dem-glo-90)       | [Copernicus DEM](https://registry.opendata.aws/copernicus-dem/)                                                                   | [cop-dem](https://github.com/stactools-packages/cop-dem)     | ~ 26,000      |
| [`landsat-c2-l2`](https://earth-search.aws.element84.com/v1/collections/landsat-c2-l2)         | [USGS Landsat](https://registry.opendata.aws/usgs-landsat/)                                                                       | [landsat](https://github.com/stactools-packages/landsat)     | ~8.5 million  |
| [`naip`](https://earth-search.aws.element84.com/v1/collections/naip)                           | [NAIP](https://registry.opendata.aws/naip/)                                                                                       | [naip](https://github.com/stactools-packages/naip)           | ~1.2 million  |
| [`sentinel-1-grd`](https://earth-search.aws.element84.com/v1/collections/sentinel-1-grd)       | [Sentinel-1 GRD](https://registry.opendata.aws/sentinel-1/)                                                                       | [sentinel1](https://github.com/stactools-packages/sentinel1) | ~2.7 million  |
| [`sentinel-2-c1-l2a`](https://earth-search.aws.element84.com/v1/collections/sentinel-2-c1-l2a) | tbd                                                                                                                               | [sentinel2](https://github.com/stactools-packages/sentinel2) | ~15 million   |
| [`sentinel-2-l1c`](https://earth-search.aws.element84.com/v1/collections/sentinel-2-l1c)       | [Sentinel-2](https://registry.opendata.aws/sentinel-2/)                                                                           | [sentinel2](https://github.com/stactools-packages/sentinel2) | ~21.7 million |
| [`sentinel-2-l2a`](https://earth-search.aws.element84.com/v1/collections/sentinel-2-l2a)       | [Sentinel-2](https://registry.opendata.aws/sentinel-2/) and [Sentinel-2 COGS](https://registry.opendata.aws/sentinel-2-l2a-cogs/) | [sentinel2](https://github.com/stactools-packages/sentinel2) | ~21.7 million |

### Copernicus DEM (cop-dem-glo-30 and cop-dem-glo-90)

The Copernicus DEM data is available in two different Collections with the only difference being the
resolution: 30 meter vs 90 meter.

### Landsat Collection 2 Level-2 (landsat-c2-l2)

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

### NAIP (naip)

NAIP data is released by year and by each individual state. Most states have a 2-3 year release cycle, although North Dakota
is released yearly. Occasionally, some data cannot be collected for a given year due to
weather or snow, so the data is collected in the following year. For example, some areas
of New York for the NAIP collection year 2021 were not collected until 2022, so their
`datatime` reflect 2022 instead of 2021. To find data for a specific NAIP year, use a
Query Extension filter on the the STAC Item property `naip:year` for the NAIP year and
`naip:state` for the state.

Prior to 2021, NAIP was only collected for the Contiguous United States (CONUS) only,
however, starting in 2021, Hawaii is also included. Puerto Rico and the US Virgin
Islands were also collected for 2021, but have not yet been published.

### Sentinel-1 (sentinel-1-grd)

The Sentinel-1 data is the Ground Range Detected amplitude-only data. Sentinel-1 is not distributed as gridded data, so the
images can vary substantially in coverage and size. The original footprints of the scenes were found to not be very
accurate, so as part of the indexing process for Earth Search, new footprints are generated.

### Sentinel-2 L1C and L2A (sentinel-2-l1c and sentinel-2-l2a)

The Sentinel-2 Collections represent the vast majority of Items stored in Earth Search, especially since both the Level-1 and
Level-2 are included. The Level-1 data is the original JPEG 2000 files, and the Level-2 data includes a set of assets for the
original JPEG 2000 files, as well as a set of assets for the Cloud-Optimized GeoTIFF (COG) versions.

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

### Sentinel-2 Collection 1 L2A (sentinel-2-c1-l2a)

The Sentinel-2 Collection 1 L2A (sentinel-2-c1-l2a) is intended to eventually
replace the other Sentinel-2 L2A collection (sentinel-2-l2a). Items in this collection
only contain assets referring to COGs of data processed to at least baseline 5.0.
However, as of April 2024, the ESA reprocessing of the entire archive to baseline
5.0 is incomplete, so the time periods of Nov 2016 to Nov 2019 and 2022 are missing.
It is unknown when this process will be completed.

## Contributions

The Earth Search API is made available as a best effort by Element 84. While it is not possible for the public to
aid in the operation of the pipeline, the creation of the STAC metadata uses repos from the open-source
[stactools-packages](https://github.com/stactools-packages) organization. PRs are welcome to these repositories which
may eventually make their way into Earth Search depending on their backwards compatability.

The best way to contribute is to provide feedback via issues in this repository.

## Updates

### May 31, 2023

- Earth Search STAC API v1 released.
