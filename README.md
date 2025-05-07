
<!-- README.md is generated from README.Rmd. Please edit that file -->

# zarrquet

Example netcdf created with

``` r
gdalmdimtranslate -co FORMAT=NC4 -co ARRAY:COMPRESS=DEFLATE \
/vsicurl/https://thredds.nci.org.au/thredds/fileServer/gb6/BRAN/BRAN2020/daily/ocean_salt_1993_01.nc \
-scaleaxes "Time(4),st_ocean(4),yt_ocean(4),xt_ocean(4)" example4d.nc
```

``` r
## ignore nv and st_edges_ocean
ncmeta::nc_dims("example4d.nc")
#> # A tibble: 6 × 4
#>      id name           length unlim
#>   <int> <chr>           <dbl> <lgl>
#> 1     0 Time                7 FALSE
#> 2     1 nv                  2 FALSE
#> 3     2 st_edges_ocean     52 FALSE
#> 4     3 st_ocean           12 FALSE
#> 5     4 xt_ocean          900 FALSE
#> 6     5 yt_ocean          375 FALSE

## as in python we have
#ds.salt.data.shape
#(7, 12, 375, 900)
#ds.salt.data.chunks
#(4, 6, 188, 450)
```

Virtualize in Python

``` python
import virtualizarr
ds = virtualizarr.open_virtual_mfdataset("example4d.nc")
ds.attrs = {} ## bad encoding of attributes so we drop 
ds.virtualize.to_kerchunk("example4d.parquet", format = "parquet")
```

Also see how the chunks are keyed by chunk-in-array index:

    ds.salt.data.manifest.keys()
    #dict_keys(['0.0.0.0', '0.0.0.1', '0.0.1.0', '0.0.1.1', '0.1.0.0', '0.1.0.1', '0.1.1.0', '0.1.1.1', '1.0.0.0', '1.0.0.1', '1.0.1.0', '1.0.1.1', '1.1.0.0', '1.1.0.1', '1.1.1.0', '1.1.1.1'])

Look at the virtual table (the default record size is 1e5 so I just
isolate to the rows that are valid). The ‘path’ is the literal file
input in-situ (can be remapped with the API at creation time).

<https://virtualizarr.readthedocs.io/en/latest/generated/virtualizarr.accessor.VirtualiZarrDatasetAccessor.rename_paths.html#virtualizarr.accessor.VirtualiZarrDatasetAccessor.rename_paths>

``` r
d <- arrow::open_dataset("example4d.parquet/salt/refs.0.parq") |> dplyr::collect()
d[!is.na(d$path), ]
#> # A tibble: 16 × 4
#>    path                 offset    size        raw
#>    <chr>                 <int>   <int> <arrw_bnr>
#>  1 /4dnc/example4d.nc    55531 1934051       NULL
#>  2 /4dnc/example4d.nc  1989582 2107082       NULL
#>  3 /4dnc/example4d.nc  7435080 1068653       NULL
#>  4 /4dnc/example4d.nc  8503733 1793109       NULL
#>  5 /4dnc/example4d.nc  4096664 1640701       NULL
#>  6 /4dnc/example4d.nc  5737365 1697715       NULL
#>  7 /4dnc/example4d.nc 10296842  765912       NULL
#>  8 /4dnc/example4d.nc 11062754 1330360       NULL
#>  9 /4dnc/example4d.nc 12393114 1458242       NULL
#> 10 /4dnc/example4d.nc 13851356 1592023       NULL
#> 11 /4dnc/example4d.nc 15443379  803851       NULL
#> 12 /4dnc/example4d.nc 16247230 1348401       NULL
#> 13 /4dnc/example4d.nc 17595631 1240646       NULL
#> 14 /4dnc/example4d.nc 18836277 1286090       NULL
#> 15 /4dnc/example4d.nc 20122367  580396       NULL
#> 16 /4dnc/example4d.nc 20702763 1005448       NULL
```

I strongly think the array key indexes should be in that table as well
but hey, there’s a strong move to Icechunk instead of this (or kerchunk
json).

The full encoding and metadata is in

``` r
## we still get weird encoding from python->json so sub out inline NaN
str(jsonlite::fromJSON(gsub("NaN", "\"NaN\"", readr::read_file("example4d.parquet/.zmetadata"))))
#> List of 2
#>  $ metadata   :List of 24
#>   ..$ .zattrs               : Named list()
#>   ..$ .zgroup               :List of 1
#>   .. ..$ zarr_format: int 2
#>   ..$ Time/.zarray          :List of 10
#>   .. ..$ shape              : int 7
#>   .. ..$ chunks             : int 7
#>   .. ..$ fill_value         : NULL
#>   .. ..$ order              : chr "C"
#>   .. ..$ filters            : NULL
#>   .. ..$ dimension_separator: chr "."
#>   .. ..$ compressor         : NULL
#>   .. ..$ attributes         : Named list()
#>   .. ..$ zarr_format        : int 2
#>   .. ..$ dtype              : chr "<f8"
#>   ..$ Time/.zattrs          :List of 8
#>   .. ..$ bounds           : chr "Time_bounds"
#>   .. ..$ calendar_type    : chr "GREGORIAN"
#>   .. ..$ cartesian_axis   : chr "T"
#>   .. ..$ long_name        : chr "Time"
#>   .. ..$ units            : chr "days since 1979-01-01"
#>   .. ..$ calendar         : chr "GREGORIAN"
#>   .. ..$ _FillValue       : chr "NaN"
#>   .. ..$ _ARRAY_DIMENSIONS: chr "Time"
#>   ..$ Time_bounds/.zarray   :List of 10
#>   .. ..$ shape              : int [1:2] 7 2
#>   .. ..$ chunks             : int [1:2] 7 2
#>   .. ..$ fill_value         : num 1e+20
#>   .. ..$ order              : chr "C"
#>   .. ..$ filters            : NULL
#>   .. ..$ dimension_separator: chr "."
#>   .. ..$ compressor         :List of 2
#>   .. .. ..$ id         : chr "shuffle"
#>   .. .. ..$ elementsize: int 8
#>   .. ..$ attributes         : Named list()
#>   .. ..$ zarr_format        : int 2
#>   .. ..$ dtype              : chr "<f8"
#>   ..$ Time_bounds/.zattrs   :List of 5
#>   .. ..$ long_name        : chr "Time axis boundaries"
#>   .. ..$ missing_value    : num 1e+20
#>   .. ..$ units            : chr "days"
#>   .. ..$ _FillValue       : chr "QIy1eB2vFUQ="
#>   .. ..$ _ARRAY_DIMENSIONS: chr [1:2] "Time" "nv"
#>   ..$ average_DT/.zarray    :List of 10
#>   .. ..$ shape              : int 7
#>   .. ..$ chunks             : int 7
#>   .. ..$ fill_value         : num 1e+20
#>   .. ..$ order              : chr "C"
#>   .. ..$ filters            : NULL
#>   .. ..$ dimension_separator: chr "."
#>   .. ..$ compressor         :List of 2
#>   .. .. ..$ id         : chr "shuffle"
#>   .. .. ..$ elementsize: int 8
#>   .. ..$ attributes         : Named list()
#>   .. ..$ zarr_format        : int 2
#>   .. ..$ dtype              : chr "<f8"
#>   ..$ average_DT/.zattrs    :List of 5
#>   .. ..$ long_name        : chr "Length of average period"
#>   .. ..$ missing_value    : num 1e+20
#>   .. ..$ units            : chr "days"
#>   .. ..$ _FillValue       : chr "QIy1eB2vFUQ="
#>   .. ..$ _ARRAY_DIMENSIONS: chr "Time"
#>   ..$ average_T1/.zarray    :List of 10
#>   .. ..$ shape              : int 7
#>   .. ..$ chunks             : int 7
#>   .. ..$ fill_value         : num 1e+20
#>   .. ..$ order              : chr "C"
#>   .. ..$ filters            : NULL
#>   .. ..$ dimension_separator: chr "."
#>   .. ..$ compressor         :List of 2
#>   .. .. ..$ id         : chr "shuffle"
#>   .. .. ..$ elementsize: int 8
#>   .. ..$ attributes         : Named list()
#>   .. ..$ zarr_format        : int 2
#>   .. ..$ dtype              : chr "<f8"
#>   ..$ average_T1/.zattrs    :List of 5
#>   .. ..$ long_name        : chr "Start time for average period"
#>   .. ..$ missing_value    : num 1e+20
#>   .. ..$ units            : chr "days since 1979-01-01 00:00:00"
#>   .. ..$ _FillValue       : chr "QIy1eB2vFUQ="
#>   .. ..$ _ARRAY_DIMENSIONS: chr "Time"
#>   ..$ average_T2/.zarray    :List of 10
#>   .. ..$ shape              : int 7
#>   .. ..$ chunks             : int 7
#>   .. ..$ fill_value         : num 1e+20
#>   .. ..$ order              : chr "C"
#>   .. ..$ filters            : NULL
#>   .. ..$ dimension_separator: chr "."
#>   .. ..$ compressor         :List of 2
#>   .. .. ..$ id         : chr "shuffle"
#>   .. .. ..$ elementsize: int 8
#>   .. ..$ attributes         : Named list()
#>   .. ..$ zarr_format        : int 2
#>   .. ..$ dtype              : chr "<f8"
#>   ..$ average_T2/.zattrs    :List of 5
#>   .. ..$ long_name        : chr "End time for average period"
#>   .. ..$ missing_value    : num 1e+20
#>   .. ..$ units            : chr "days since 1979-01-01 00:00:00"
#>   .. ..$ _FillValue       : chr "QIy1eB2vFUQ="
#>   .. ..$ _ARRAY_DIMENSIONS: chr "Time"
#>   ..$ nv/.zarray            :List of 10
#>   .. ..$ shape              : int 2
#>   .. ..$ chunks             : int 2
#>   .. ..$ fill_value         : NULL
#>   .. ..$ order              : chr "C"
#>   .. ..$ filters            : NULL
#>   .. ..$ dimension_separator: chr "."
#>   .. ..$ compressor         : NULL
#>   .. ..$ attributes         : Named list()
#>   .. ..$ zarr_format        : int 2
#>   .. ..$ dtype              : chr "<f8"
#>   ..$ nv/.zattrs            :List of 2
#>   .. ..$ _FillValue       : chr "NaN"
#>   .. ..$ _ARRAY_DIMENSIONS: chr "nv"
#>   ..$ salt/.zarray          :List of 10
#>   .. ..$ shape              : int [1:4] 7 12 375 900
#>   .. ..$ chunks             : int [1:4] 4 6 188 450
#>   .. ..$ fill_value         : num -32768
#>   .. ..$ order              : chr "C"
#>   .. ..$ filters            :'data.frame':   1 obs. of  5 variables:
#>   .. .. ..$ id    : chr "fixedscaleoffset"
#>   .. .. ..$ scale : num 596
#>   .. .. ..$ offset: num 45
#>   .. .. ..$ dtype : chr "<f8"
#>   .. .. ..$ astype: chr "<i2"
#>   .. ..$ dimension_separator: chr "."
#>   .. ..$ compressor         :List of 2
#>   .. .. ..$ id         : chr "shuffle"
#>   .. .. ..$ elementsize: int 2
#>   .. ..$ attributes         : Named list()
#>   .. ..$ zarr_format        : int 2
#>   .. ..$ dtype              : chr "<f8"
#>   ..$ salt/.zattrs          :List of 11
#>   .. ..$ cell_methods     : chr "time: mean"
#>   .. ..$ coordinates      : chr "geolon_t geolat_t"
#>   .. ..$ long_name        : chr "Practical Salinity"
#>   .. ..$ missing_value    : int -32768
#>   .. ..$ packing          : int 4
#>   .. ..$ standard_name    : chr "sea_water_salinity"
#>   .. ..$ time_avg_info    : chr "average_T1,average_T2,average_DT"
#>   .. ..$ valid_range      : int [1:2] -32767 32767
#>   .. ..$ units            : chr "psu"
#>   .. ..$ _FillValue       : int -32768
#>   .. ..$ _ARRAY_DIMENSIONS: chr [1:4] "Time" "st_ocean" "yt_ocean" "xt_ocean"
#>   ..$ st_edges_ocean/.zarray:List of 10
#>   .. ..$ shape              : int 52
#>   .. ..$ chunks             : int 52
#>   .. ..$ fill_value         : NULL
#>   .. ..$ order              : chr "C"
#>   .. ..$ filters            : NULL
#>   .. ..$ dimension_separator: chr "."
#>   .. ..$ compressor         : NULL
#>   .. ..$ attributes         : Named list()
#>   .. ..$ zarr_format        : int 2
#>   .. ..$ dtype              : chr "<f8"
#>   ..$ st_edges_ocean/.zattrs:List of 6
#>   .. ..$ cartesian_axis   : chr "Z"
#>   .. ..$ long_name        : chr "tcell zstar depth edges"
#>   .. ..$ positive         : chr "down"
#>   .. ..$ units            : chr "meters"
#>   .. ..$ _FillValue       : chr "NaN"
#>   .. ..$ _ARRAY_DIMENSIONS: chr "st_edges_ocean"
#>   ..$ st_ocean/.zarray      :List of 10
#>   .. ..$ shape              : int 12
#>   .. ..$ chunks             : int 12
#>   .. ..$ fill_value         : NULL
#>   .. ..$ order              : chr "C"
#>   .. ..$ filters            : NULL
#>   .. ..$ dimension_separator: chr "."
#>   .. ..$ compressor         : NULL
#>   .. ..$ attributes         : Named list()
#>   .. ..$ zarr_format        : int 2
#>   .. ..$ dtype              : chr "<f8"
#>   ..$ st_ocean/.zattrs      :List of 7
#>   .. ..$ cartesian_axis   : chr "Z"
#>   .. ..$ edges            : chr "st_edges_ocean"
#>   .. ..$ long_name        : chr "tcell zstar depth"
#>   .. ..$ positive         : chr "down"
#>   .. ..$ units            : chr "meters"
#>   .. ..$ _FillValue       : chr "NaN"
#>   .. ..$ _ARRAY_DIMENSIONS: chr "st_ocean"
#>   ..$ xt_ocean/.zarray      :List of 10
#>   .. ..$ shape              : int 900
#>   .. ..$ chunks             : int 900
#>   .. ..$ fill_value         : NULL
#>   .. ..$ order              : chr "C"
#>   .. ..$ filters            : NULL
#>   .. ..$ dimension_separator: chr "."
#>   .. ..$ compressor         : NULL
#>   .. ..$ attributes         : Named list()
#>   .. ..$ zarr_format        : int 2
#>   .. ..$ dtype              : chr "<f8"
#>   ..$ xt_ocean/.zattrs      :List of 6
#>   .. ..$ cartesian_axis      : chr "X"
#>   .. ..$ domain_decomposition: int [1:4] 1 3600 1 1800
#>   .. ..$ long_name           : chr "tcell longitude"
#>   .. ..$ units               : chr "degrees_E"
#>   .. ..$ _FillValue          : chr "NaN"
#>   .. ..$ _ARRAY_DIMENSIONS   : chr "xt_ocean"
#>   ..$ yt_ocean/.zarray      :List of 10
#>   .. ..$ shape              : int 375
#>   .. ..$ chunks             : int 375
#>   .. ..$ fill_value         : NULL
#>   .. ..$ order              : chr "C"
#>   .. ..$ filters            : NULL
#>   .. ..$ dimension_separator: chr "."
#>   .. ..$ compressor         : NULL
#>   .. ..$ attributes         : Named list()
#>   .. ..$ zarr_format        : int 2
#>   .. ..$ dtype              : chr "<f8"
#>   ..$ yt_ocean/.zattrs      :List of 6
#>   .. ..$ cartesian_axis      : chr "Y"
#>   .. ..$ domain_decomposition: int [1:4] 1 1500 1 150
#>   .. ..$ long_name           : chr "tcell latitude"
#>   .. ..$ units               : chr "degrees_N"
#>   .. ..$ _FillValue          : chr "NaN"
#>   .. ..$ _ARRAY_DIMENSIONS   : chr "yt_ocean"
#>  $ record_size: int 100000
```
