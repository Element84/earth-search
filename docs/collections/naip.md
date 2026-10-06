# NAIP (naip)

- Collection: [`naip`](https://earth-search.aws.element84.com/v1/collections/naip)
- Public dataset: [NAIP](https://registry.opendata.aws/naip/)
- stactools package: [naip](https://github.com/stactools-packages/naip)
- Number of Items: ~1.2 million

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

Back to the [Earth Search README](../../README.md#collections).
