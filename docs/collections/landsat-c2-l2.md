# Landsat Collection 2 Level-2 (landsat-c2-l2)

- Collection: [`landsat-c2-l2`](https://earth-search.aws.element84.com/v1/collections/landsat-c2-l2)
- Public dataset: [USGS Landsat](https://registry.opendata.aws/usgs-landsat/)
- stactools package: [landsat](https://github.com/stactools-packages/landsat)
- Number of Items: ~8.5 million

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

## Known Issues

- When USGS reprocesses an Item, the newly processed datafiles replace the old Items in Earth Search (because the ID
is the same). However there are some historical scenes in the index that do not point to the most recently processed
scene, which means the assets contain dead links.

Back to the [Earth Search README](../../README.md#collections).
