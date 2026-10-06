# Earth Search STAC API <!-- omit from toc -->

- [General Notes](#general-notes)
  - [Extensions Used](#extensions-used)
  - [s3 vs http URLs](#s3-vs-http-urls)
  - [SNS Notifications of Items](#sns-notifications-of-items)
- [Collections](#collections)
- [Contributions](#contributions)
- [Updates](#updates)
  - [Earth Search v2 (Beta)](#earth-search-v2-beta)
  - [May 31, 2023](#may-31-2023)

This README contains information on [Element 84](https://element84.com)'s [Earth Search STAC API](https://earth-search.aws.element84.com/v1).
Earth Search is a free-to-use SpatioTemporal Asset Catalog (STAC) API containing an index of geospatial data collections
available on the [AWS Registry of Open Data](https://aws.amazon.com/earth/) (RODA).

This public API does not come with any guaranteed service. If you are using Earth Search in production
and are interested in a private data catalog that can incorporate both public and private data sources,
[please reach out](https://www.element84.com/work-with-us). Earth Search is powered by [FilmDrop](https://element84.com/filmdrop), a collection of
open-source applications and libraries for geospatial data ingest and cataloging supported by [Element 84](https://element84.com).

To stay up to date with any new changes or datasets to the Earth Search API, please sign up for the [mailing list](http://eepurl.com/irrwDo), you can
also review [the mailing list archive](https://us13.campaign-archive.com/home/?u=a7a7fcb1ce46c4d001fc76289\&id=38266ef009).
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
- start\_datetime (String): The STAC Item `start_datetime` value, a string in RFC 3339 format, if that field exists; otherwise, the `datetime` value.
- end\_datetime (String): The STAC Item `end_datetime` value, a string in RFC 3339 format, if that field exists; otherwise, the `datetime` value.
- bbox.ll\_lon (Number): The lower left (southwest) longitude of the Item bounding box
- bbox.ll\_lat (Number): The lower left (southwest) latitude of the Item bounding box
- bbox.ur\_lon (Number): The upper right (northeast) longitude of the Item bounding box
- bbox.ur\_lat (Number): The upper right (northeast) latitude of the Item bounding box
- cloud\_cover (Number): The STAC Item `eo:cloud_cover` value, a floating point value between 0 and 100.
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

## Collections

The STAC Collections in the API each have their own page with specific notes on the dataset including
any known issues. Please file an issue in this repository for any new issues, such as incorrect metadata.
Some Collections do have missing Items due to several issues and these are being worked on.

| Collection                                                   | Public Dataset                                                                                                                    | stactools package                                            | # of Items     |
| ------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------ | -------------- |
| [`cop-dem-glo-30`](docs/collections/cop-dem-glo-30.md)       | [Copernicus DEM](https://registry.opendata.aws/copernicus-dem/)                                                                   | [cop-dem](https://github.com/stactools-packages/cop-dem)     | \~ 26,000      |
| [`cop-dem-glo-90`](docs/collections/cop-dem-glo-90.md)       | [Copernicus DEM](https://registry.opendata.aws/copernicus-dem/)                                                                   | [cop-dem](https://github.com/stactools-packages/cop-dem)     | \~ 26,000      |
| [`landsat-c2-l2`](docs/collections/landsat-c2-l2.md)         | [USGS Landsat](https://registry.opendata.aws/usgs-landsat/)                                                                       | [landsat](https://github.com/stactools-packages/landsat)     | \~8.5 million  |
| [`naip`](docs/collections/naip.md)                           | [NAIP](https://registry.opendata.aws/naip/)                                                                                       | [naip](https://github.com/stactools-packages/naip)           | \~1.2 million  |
| [`sentinel-1-grd`](docs/collections/sentinel-1-grd.md)       | [Sentinel-1 GRD](https://registry.opendata.aws/sentinel-1/)                                                                       | [sentinel1](https://github.com/stactools-packages/sentinel1) | \~2.7 million  |
| [`sentinel-2-c1-l2a`](docs/collections/sentinel-2-l2a.md)    | tbd                                                                                                                               | [sentinel2](https://github.com/stactools-packages/sentinel2) | \~15 million   |
| [`sentinel-2-l1c`](docs/collections/sentinel-2-l1c.md)       | [Sentinel-2](https://registry.opendata.aws/sentinel-2/)                                                                           | [sentinel2](https://github.com/stactools-packages/sentinel2) | \~21.7 million |
| [`sentinel-2-l2a`](docs/collections/sentinel-2-l2a.md)       | [Sentinel-2](https://registry.opendata.aws/sentinel-2/) and [Sentinel-2 COGS](https://registry.opendata.aws/sentinel-2-l2a-cogs/) | [sentinel2](https://github.com/stactools-packages/sentinel2) | \~21.7 million |

## Contributions

The Earth Search API is made available as a best effort by Element 84. While it is not possible for the public to
aid in the operation of the pipeline, the creation of the STAC metadata uses repos from the open-source
[stactools-packages](https://github.com/stactools-packages) organization. PRs are welcome to these repositories which
may eventually make their way into Earth Search depending on their backwards compatability.

The best way to contribute is to provide feedback via issues in this repository.

## Updates

### Earth Search v2 (Beta)

Earth Search STAC API v2 is currently in beta and may change before general release.

Collection-specific v2 changes are documented on each collection's page:

- [Sentinel-2 L2A (v1 `sentinel-2-c1-l2a` to v2 `sentinel-2-l2a`)](docs/collections/sentinel-2-l2a.md#changes-from-v1-sentinel-2-c1-l2a-to-v2-sentinel-2-l2a)

### May 31, 2023

- Earth Search STAC API v1 released.
