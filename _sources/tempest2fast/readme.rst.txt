TempEst 2-FAST
==============

TempEst 2/SCHEMA is “TEMPerature ESTimation, version 2, using Seasonal
Conditions Historical Expectation with Modeled daily Anomaly”. It
estimates stream water temperature, for streams of any size, using a
data-driven model based on satellite remote sensing data. TempEst 2-FAST
is optimized for large-scale gridded applications, rather than discrete
points. This involves two tradeoffs:

-  Build-in data retrieval and native processing, rather than Google
   Earth Engine. For a small set of points, this is slower and
   resource-intensive. However, it means you don’t run into GEE memory
   and runtime limits, and it lets all the processing happen in one
   place. That way you also don’t have to set up GEE if you don’t have
   it. I have found that, while GEE is a wonderful tool provided for
   free to academic researchers, it gets a bit unwieldy when you move
   from thousands of points to tens or hundreds of thousands.
-  Gridded, not discrete points. This means you can’t specify exact
   sites, or distinguish between several small streams that are all in a
   single 1-km pixel. However, it means the whole thing can be
   efficiently downloaded and processed as an array, without duplicating
   effort across adjacent points.

In short, if you need data for a few thousand sites, use the original
TempEst 2. If you need contiguous, high-resolution data over a large
region - with at least 10,000 or so river-kilometers - use TempEst
2-FAST. It’s designed to take advantage of xarray’s Dask-based design
for larger-than-memory datasets and tested on a desktop (for a small
test region), so it should work on a personal computer, but of course
for large-scale datasets you’re better off with HPC or similar, and it
should parallelize automatically.

To avoid duplicating effort, the spatial components of the coefficient
models are exported from TempEst 2 as kilometer-resolution rasters, as
well as the model coefficients as a CSV. TempEst 2-FAST handles the
implementation.

The exported surfaces required for prediction ``SpatialModels.nc`` and
the model coefficients ``FixedModel.csv`` are included in the GitHub
repository, along with the elevation (``conus_1km.nc``) and hydrography
(``Rivers.nc``) datasets. The full river temperature dataset is
forthcoming.

Usage Guide
-----------

Install from the Python Package Index: ``pip install tempest2fast``. A
full reference may be found at
`RiverTempest.org <https://rivertempest.org/tempest2fast/tempest2fast.html>`__.
If you are viewing this page on PyPI, all downloads provided with the
model may be found at the `GitHub
repository <https://github.com/mines-ciroh/TempEst2FAST/>`__.

Dependencies:

-  xarray
-  pandas
-  numpy
-  rioxarray
-  earthaccess
-  pynldas2
-  dask
-  s3fs
-  rtseason

Generating Predictions
~~~~~~~~~~~~~~~~~~~~~~

Since TempEst 2-FAST doesn’t handle training, its prediction functions
require paths for the spatial surfaces and fixed model coefficients.

The other arguments to the prediction function are the paths for the
land cover, hydrography, elevation (single files), humidity, and LST
(multifile) datasets, or the pre-loaded datasets.

TempEst 2-FAST will return an xarray dataset containing:

-  temp.mod (modeled daily mean temperature; Celsius)
-  temp.doy (modeled day-of-year mean temperature; Celsius)
-  temp.anom (modeled temperature anomaly relative to day-of-year mean;
   Celsius)
-  temp.plus (modeled maximum temperature relative to day-of-year mean;
   Celsius)
-  temp.max (modeled maximum daily temperature; Celsius)

``temp.mod`` = ``temp.doy`` + ``temp.anom``.

These five data arrays are the final output, and can be used to assess
actual estimated temperature (``temp.mod``, ``temp.max``); typical
seasonal conditions (``temp.doy``), such as to assess general long-term
behavior; and departure from typical seasonal conditions (``temp.anom``,
``temp.plus``), such as to assess heat waves or other extremes.

In my testing, memory requirements and compute time are sublinear with
area of interest (but not with period of record), though out-of-memory
computing is imperfect (read: it *will* crash from lack of memory).
Therefore, I recommend running the largest area your resources support
at a time, but manually chunking the run may be necessary. The top-level
``run.py`` includes an example of breaking the CONUS-wide data into
subsets for efficient processing. A 1x1-degree, 24-year run (10,000
total pixels, ~600 with rivers) takes 12.5 GB and 2.5 hours. (Note that
600 points is well within the range where regular TempEst 2 would be the
preferred tool.)

Very Quick Version
^^^^^^^^^^^^^^^^^^

``tempest2fast.full_run(fixed, rivers, elev, lcov, lst, humidity, spat, bbox).to_netcdf(output)``.
All arguments are paths to the required datasets (single files for most;
multi-file paths with wildcards for humidity and LST, like
“humidity*.nc”) or the output file location.

Chunked Example (run.py)
^^^^^^^^^^^^^^^^^^^^^^^^

::

   from tempest2fast import full_run
   import tempest2fast.prediction as prediction
   from tempest2fast.defaults import bbox as full_bbox
   import pandas as pd
   from glob import glob
   import sys
   from math import ceil
   inp_base = "/scratch/dphilippus/FAST/"
   out_base = inp_base + "outputs/"

   fixed = inp_base + "FixedModel.csv"
   rivers = inp_base + "Rivers.nc"
   lcov = inp_base + "lcov.nc"
   elev = inp_base + "conus_1km.nc"
   spat = inp_base + "SpatialModels.nc"
   lst_base = inp_base + "LSTRch/"
   hum_base = inp_base + "HumidityRch/"
   lst = glob(lst_base + "modis_lst_*")
   humidity = glob(hum_base + "humidity_*")

   logfile = "runlog.txt"

   if __name__ == "__main__":
       # Run in NxN-degree chunks to limit memory use.
       args = sys.argv
       if len(args) < 2:
           desc = "all"
           N = 0
       if len(args) >= 2:
           N = int(args[1])
       window_size = 1
       if len(args) >= 3:
           window_size = int(args[2])
       desc = f"_{window_size}deg_no{N}"
       outputco = out_base + f"cochunks/coefs_{desc}.nc"
       outputts = out_base + f"chunks/ts_{desc}.nc"
       # Build windows. Default bbox is 60x25.
       # Thus,
       # 1x1 -> 1500
       # 5x5 -> 60
       # 2x2 -> 390
       # 3x3 -> 180
       # 4x4 -> 105
       columns = ceil((full_bbox[2] - full_bbox[0]) / window_size)
       rows = ceil((full_bbox[3] - full_bbox[1]) / window_size)
       # Position:
       # Which *row* is the number of complete columns it covers,
       # and which *column* is mod rows.
       irow = N // columns
       icol = N % rows
       bbox = (
               full_bbox[0] + icol * window_size,
               full_bbox[1] + irow * window_size,
               full_bbox[0] + (icol + 1) * window_size,
               full_bbox[1] + (irow + 1) * window_size
               )
       print(f"Running index {N} with bbox {bbox}")
       full_run(fixed, rivers, elev, lcov, lst, humidity, spat, bbox, outputco, logfile).to_netcdf(outputts)

Retrieving Prediction Data
~~~~~~~~~~~~~~~~~~~~~~~~~~

Suitable prediction data can be retrieved in any way, but TempEst 2-FAST
has built-in code to download and process some of the required data
(MODIS LST, NLDAS2 humidity, Copernicus global DEM). Land cover from ESA
WorldCover and (if outside the CONUS) MERIT Hydro hydrography must be
downloaded manually, but TempEst 2-FAST can take it from there. TempEst
2-FAST can automatically download NLDAS-2, but the runtime is excessive
for a large dataset, so manually downloading humidity is recommended.
Processing involves reprojecting the gridded datasets into a consistent
format, resampling, and creating an array of river IDs. All datasets are
converted to a 0.01-degree grid with centroids at each 0.01 degrees.
Inputs are not, in principle, dataset-specific, but be wary of
differences in dataset estimates, which are likely embedded into the
model coefficients to some degree. I’d recommend retraining the TempEst
2 model if you use different input datasets.

For ESA WorldCover,
`download <https://worldcover2021.esa.int/downloader>`__ and unzip the
macrotiles corresponding to your area of interest. TempEst 2-FAST then
requires the extracted directory as an argument and will take it from
there.

For NLDAS-2 humidity, generate the subset download links
`here <https://disc.gsfc.nasa.gov/datasets/NLDAS_FORB0125_H_2.0/summary?keywords=NLDAS>`__,
then follow the wget download instructions
`here <https://disc.gsfc.nasa.gov/information/howto?title=How%20to%20Access%20GES%20DISC%20Data%20Using%20wget%20and%20curl>`__.
This is far faster than using pynldas2, and can download about 2 days
per minute. After that, use ``humidity.process_downloaded`` to convert
the data into the appropriate format. However, for smaller areas,
``humidity.process_days`` (via PyNLDAS2) may be appropriate.

The river dataset uses river widths from `MERIT
Hydro <https://hydro.iis.u-tokyo.ac.jp/~yamadai/MERIT_Hydro/>`__. A
GeoTIFF of river widths for the CONUS was downloaded using Google Earth
Engine and is included. For other areas, you can download a suitable
river width dataset directly from MERIT Hydro or using GEE.

TempEst 2-FAST ships with pre-processed elevation and land cover for the
CONUS, since those (being static) are reasonably small (a few hundred
megabytes). LST needs to be downloaded locally, since the default
coverage (~24 years over the CONUS) is about a terabyte of data. I
recommend running ``lst.build_multids`` directly in a terminal, using
the ``start`` argument to pick up where you left off; you should not
need to directly access any other functions from the ``lst`` module, and
manual downloading is not necessary. Initial running must be interactive
since you will be asked for EarthData login information.

Note that, despite lazy loading, reading through large input files is a
major bottleneck, so I recommend only downloading the areas you intend
to process.

Training a Model
~~~~~~~~~~~~~~~~

See `TempEst 2 <https://github.com/mines-ciroh/TempEst2>`__. Then run
the extra step in ``demo.Rmd`` there to export surfaces and
coefficients. One entry of the Variable column will be blank; this
should be set to ``AutumnWinter``. To avoid redundancy, TempEst 2-FAST
does not handle model training.

Citation and More Information
-----------------------------

If you use this model in your research, please cite TempEst 2.

The model design is based on Philippus, Corona, Schneider, Rust, and
Hogue, 2025, “Satellite-Based Spatial-Statistical Modeling of Daily
Stream Water Temperatures at the CONUS Scale”, *Journal of Hydrology*.
This paper also contains a detailed description of model design and
performance characteristics.

The seasonality component is based on the “three-sine” stream annual
temperature cycle function described in: Philippus, Corona, and Hogue,
2024, “Improved annual temperature cycle function for stream seasonal
thermal regimes”, *JAWRA*, https://doi.org/10.1111/1752-1688.13228.

Full training datasets, pre-trained models, and a knitted model
validation notebook PDF are available in CUAHSI HydroShare: `TempEst 2
Development Data: Observed Stream Temperature, Covariates, Performance
Data, and Analysis
Notebooks <https://www.hydroshare.org/resource/a8b243957f7946e388d10ab206990675/>`__.

The Python package implementing TempEst 2-FAST is available on the
Python Package Index:
`pypi.org/project/tempest2fast <https://pypi.org/project/tempest2fast>`__.
