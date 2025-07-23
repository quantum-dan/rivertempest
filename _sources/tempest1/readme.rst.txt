TempEst 1
=========

Tempest is a river TEMPerature ESTimation model implemented in R; it
supports high-accuracy estimation of river temperatures using only
satellite-based remote sensing data. It has been validated for
publicly-available datasets covering the contiguous United States, but
should be generalizable to other regions and datasets. Detailed
performance information is available in Philippus et al. 2024 (see
`Citation <#citation>`__); briefly, median validation Root Mean Square
Error is about 1.5 C for individual predictions, and errors are normally
distributed within roughly plus or minus 4.5 C with bias of
approximately 0. Therefore, the model is suitable for statistical
analyses of hundreds or thousands of points, but should be used with
caution for individual points or reaches without validation data. The
model inputs range in resolution from 10 to 500 meters; in practice,
points closer together than a few hundred meters will likely not provide
usefully different outputs, as the data collection radius is 1000 m.

TempEst is not currently being actively developed, but further
development is possible upon request. In particular, open an Issue if
your use case would benefit from:

-  In R, having the ``predict`` method implemented for the model in
   addition to the custom prediction function
-  Having either the Google Earth Engine retrieval code or TempEst
   itself ported to Python
-  Non-GEE functions for data retrieval (e.g. direct raster retrieval
   from data provider APIs and local processing) - note that this would
   likely be much slower than GEE

Note that the model is provided with absolutely no warranty regarding
its accuracy or anything else. The above performance information is
typical behavior, not a guarantee.

Quick Start
-----------

1. Download the repository (clone or download the archive from
   Releases).
2. Install (if necessary) the R libraries “dplyr”, “tidyr”,
   “quantregForest”, and “purrr”. Versions tested:

   -  R: 4.1.2 on Windows 10 x64
   -  dplyr: 1.0.7 (via tidyverse)
   -  tidyr: 1.1.3 (via tidyverse)
   -  purrr: 0.3.4 (via tidyverse)
   -  quantregForest: 1.3 (dependency: randomForest at 4.7)
   -  For testing, data frames were generally loaded with ``read_csv``
      from readr, not the base R ``read.csv``.

3. To use the default trained model, simply download ``Tempest.RData``
   from Releases and ``load("Tempest.RData")``, which will load the
   model as ``tempest.model`` trained on ``Calibration.csv`` (note that
   the pre-trained model currently makes predictions in Kelvin; update
   pending). Alternatively, to train a model on your own dataset or
   ``Data.csv``:

   1. Load the calibration dataset ``Data.csv``, or your own, as a data
      frame (read.csv or read_csv). All quantitative inputs must be as
      numeric columns.
   2. This dataset is the first, required, argument to ``make.model()``.
      Store the output of ``make.model()`` in a variable.
   3. Optionally, add the named argument ``nthreads``
      (e.g. ``make.model(caldata, nthreads=16)``) to speed up training
      substantially. Training the default model with ``nthreads=16``
      takes approximately 20 seconds on the developer’s machine.

4. Download the prediction inputs for the points of interest. Follow the
   steps in `Data Collection <#Data-Collection>`__ to download data for
   your points; you can go to the `code
   snapshot <https://code.earthengine.google.com/d0de7accd1e10300b8e38bf5295de610>`__
   with examples and documentation or load the whole repository as
   below. The provided Earth Engine code has an example set up and
   documented.
5. Download the resulting dataset and load it as a data frame
   (``read.csv`` or ``read_csv``).
6. Run the model using ``predict.temperature()``.

   -  The first argument is the model from (3).
   -  The second argument is the data from (5).
   -  The output is a data frame of: id, ecoregion, year, time, start,
      end, temperature (predicted temperature in Celsius).

7. (Optional) Check model accuracy. The Notebook ``validation.Rmd``
   contains 5-fold cross-validation with ``Data.csv``. Otherwise, read
   in your validation dataset as a data frame; this should contain all
   the usual input columns plus a ``temperature`` (known temperature in
   Celsius) column. Use this dataset as the second argument to
   ``predict.temperature`` with the named argument ``compare=T``. The
   output will have columns ``Actual`` and ``Modeled`` in place of
   ``temperature``, which you can use for error analysis. Given this
   output as its argument, the provided function ``error.bxp()``
   (additional dependency: ``ggplot2``) generates boxplots, grouped by
   ecoregion, of each gage’s Root Mean Square Error (RMSE), Percent
   Bias, and Coefficient of Determination (R2).
8. (Optional) Predict temperature quantiles. ``predict.temperature``
   also takes the named argument ``what``,
   e.g. ``what=c(0.05, 0.5, 0.95)``, which will predict quantiles
   instead of just the median. The output columns will be named
   ``<basename>_<quantile>``, e.g. ``temperature_0.05`` or (if
   ``compare=T`` is also specified) ``Modeled_0.05``.
9. (Optional) Data visualization. The function ``plot.temperature``
   (additional dependency: ``ggplot2``) plots the prediction points on a
   grid (decimal latitude/longitude) colored by their overall average
   temperature. The argument to ``plot.temperature``, in order to have
   the columns ``lat``, ``lon``, and ``temperature``, must be the result
   of running ``predict.temperature`` with:

   -  Argument ``preserve=TRUE``
   -  Argument ``compare=FALSE`` (default)
   -  AND the data (second) argument must not have a ``temperature``
      column.

Usage
-----

The model can be trained and tested on any tabular dataset. If
``default.inputs=TRUE`` (the default), the input dataset must have the
columns
``start, end, id, time, year, temperature,avgtemp, landtemp, humidity,lat, lon, elevation,water, builtup, trees``
(model extrapolation performance, e.g. to high altitudes and the arid
Southwest, is better with the more limited inputs). Otherwise, column
names are mostly irrelevant, and it will simply use all columns that are
provided with a few exceptions. By default (but easily changed in the
code), the output column in the training data should be called
``temperature``, and a few named columns will be dropped. Otherwise the
only constraint is that the training and prediction datasets should have
the same column names. Default calibration and validation datasets,
covering CONUS stream temperatures based on USGS gages, are provided. By
default the prediction function will simply predict median temperatures,
but the ``what`` argument can be used to specify quantiles to predict,
e.g. to estimate confidence intervals. To preserve an observed column
for validation, use the ``compare=T`` argument to
``predict.temperature``.

In general, this model is well-suited to large-scale analyses of
hundreds or thousands of locations over time. It allows the user to
retrieve large amounts of data from across the contiguous United States
- that limitation being due to the default calibration dataset, not the
model implementation - with no resources other than Google Earth Engine
and with good accuracy. This will support analyses of general, regional
or national, patterns in average stream temperatures. However, for
single locations where extensive data are readily available, the model
is less accurate than detailed hydrologic models. In addition, while the
errors (being roughly normally distributed with a mean of approximately
zero) do average out at scale, any individual point may have substantial
errors, so single point predictions should be used with caution without
validation data. If single point predictions are used, we suggest using
the ``what`` argument to ``predict.temperature`` in order to quantify
the uncertainty. Uncertainty predictions are usually accurate but
slightly conservative: the gagewise median has 53% of observations in
the 50% confidence interval and 98% in the 95% confidence interval.

``Data.csv`` is the default testing and development dataset.
``Calibration.csv`` and ``Validation.csv`` are included for
reproducibility of early development work, but are not in active use for
model development.

Validation
---

The Notebook ``validation.Rmd`` will run through an extensive suite of
cross-validation tests using ``Data.csv`` (or your own dataset). Use
this notebook to fully reproduce model performance data.

Data Collection
---------------

For the default setup, we provide a `Google Earth Engine
script <https://code.earthengine.google.com/?accept_repo=users/dphilippus_mines/RST>`__
(open ``runner.js`` and edit/run as appropriate) (`code
snapshot <https://code.earthengine.google.com/d0de7accd1e10300b8e38bf5295de610>`__)
which will retrieve data in a suitable format for specified points and
time intervals. The GEE script is provided under the same terms, except
that it builds on open-source code from Ermida et al. (2020), which
should also be credited and cited in any research using it. Full
citation information for Ermida et al. is provided in the comments in
``runner.js``. Full instructions are included in the comments in
``runner.js``. (Note that the first link is useful if you wish to save
the repository to your own account; otherwise, the code snapshot is
likely more helpful.)

In our experience, the Earth Engine retrieval script takes roughly 20
minutes to 1 hour per 10,000 observations (point-months). The maximum
allowed runtime in GEE is 12 hours, but to avoid memory limitations we
generally run jobs in batches of 500 points and 1 year. Earth Engine
usually runs several jobs in parallel.

License
-------

This model is provided under the terms of the GNU General Public License
v3.0. It comes with absolutely no warranty, and you are free to use,
modify, and redistribute it as long as any derivatives are provided
under the same terms and credit is given. By using this model or any
data derived from it for research, you agree to cite Philippus et
al. 2024.

Citation
--------

The full model design and performance are documented in: Philippus,
Sytsma, Rust, and Hogue, (2024), “A Machine Learning Model for
Estimating the Temperature of Small Rivers Using Satellite-Based Spatial
Data”, in *Remote Sensing of Environment*
(https://doi.org/10.1016/j.rse.2024.114271).
