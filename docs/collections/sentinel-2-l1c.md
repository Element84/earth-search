# Sentinel-2 L1C (sentinel-2-l1c)

- Collection: [`sentinel-2-l1c`](https://earth-search.aws.element84.com/v1/collections/sentinel-2-l1c)
- Public dataset: [Sentinel-2](https://registry.opendata.aws/sentinel-2/)
- stactools package: [sentinel2](https://github.com/stactools-packages/sentinel2)
- Number of Items: ~21.7 million

The Sentinel-2 Collections represent the vast majority of Items stored in Earth Search, especially since both the Level-1 and
Level-2 are included. The Level-1 data is the original JPEG 2000 files. See also [`sentinel-2-l2a`](sentinel-2-l2a.md).

## Known Issues

- Most of the scenes crossing the antimeridian are excluded from the Catalog. This issue has been resolved in the
stactools-sentinel2 package by splitting polygons crossing the antimeridian. The historical scenes will be indexed
in the coming months.
- There are some missing Items from the Catalog because currently the L1 and L2 data are ingested together and in some cases
there is no available L1 data which causes the ingest to fail. The causes for this are being explored and we will likely move
to ingesting the L2 data regardless if L1 is available or not.

Back to the [Earth Search README](../../README.md#collections).
