TempEst 2
=========

TempEst 2/SCHEMA is “TEMPerature ESTimation, version 2, using Seasonal
Conditions Historical Expectation with Modeled daily Anomaly”. It
estimates stream water temperature, for streams of any size, using a
data-driven model based on satellite remote sensing data. This is the
Python implementation with some additional data processing features.

Note that the geostatistical implementation is similar but *not
identical* to the original (R) impementation described in Philippus et
al., 2025 because PyKrige does not have the same spatial covariance
functions available as fields and does not have MLE-based covariance
function parameter estimation.

Usage Guide
-----------

Full documentation is available at RiverTempest.org.

Generating Predictions
~~~~~~~~~~~~~~~~~~~~~~

TempEst 2 will return:

- temp.mod (modeled daily mean temperature; Celsius)
- temp.doy (modeled day-of-year mean temperature; Celsius)
- temp.anom (modeled temperature anomaly relative to day-of-year mean;
  Celsius)

``temp.mod`` = ``temp.doy`` + ``temp.anom``.

These three variables are the final output, and can be used to assess
actual estimated temperature (``temp.mod``); typical seasonal conditions
(``temp.doy``), such as to assess general long-term behavior; and
departure from typical seasonal conditions (``temp.anom``), such as to
assess heat waves or other extremes.

1. Install TempEst 2: ``pip install TempEst2``
2. Download pre-trained model pickle from `GitHub
   Releases <https://github.com/mines-ciroh/pyTE2/releases>`__ as
   ``model.pkl``
3. Download prediction data by any suitable means, such as the
   `reference implementation’s <github.com/mines-ciroh/tempest2>`__
   Google Earth Engine data retrieval script, as ``data.csv``. Required
   columns: id, date, lst, humidity, lat, lon, elevation, grassland,
   shrubland, barren, water.
4. ``predictions = tempest2.predict('model.pkl', 'data.csv')``

Relevant Functionality
^^^^^^^^^^^^^^^^^^^^^^

The above-described ``predict`` function handles the entire prediction
process. There are several key functions that may be relevant on their
own and to fitting a model:

- Spatial prediction:

  - ``tempest2.spatial.to_pickle`` and ``.from_pickle``: store and
    retrieve geospatial models in Pickle format.
  - ``tempest2.build_model``: train a geospatial model from timeseries
    data.
  - ``tempest2.predict_coefs`` and ``.predict_timeseries``: predict site
    coefficients and full timeseries, respectively. ``predict_coefs``
    requires summarized inputs, while ``.predict_timeseries`` uses a
    full timeseries input.

- Single site:

  - ``tempest2.full_fit``: fit TempEst 2 coefficients from a site
    timeseries. Unlike the reference implementation, this begins with a
    direct solution, then iteratively optimizes.
  - ``tempest2.timeseries``: predict a timeseries for a single site
    using a timeseries input and specified coefficients.
  - ``tempest2.Model``: LibSCHEMA model implementation

- ``tempest2.full_summary``: generate prediction data from an input
  timeseries.

Citation and More Information
-----------------------------

If you use this model in your research, please cite TempEst 2 (Philippus
et al., 2025).

Reference implementation in R and data retrieval tools for Google Earth
Engine: https://github.com/mines-ciroh/tempest2

The model design is based on Philippus, Corona, Schneider, Rust, and
Hogue, 2025, “Satellite-Based Spatial-Statistical Modeling of Daily
Stream Water Temperatures at the CONUS Scale”, *Journal of Hydrology*,
https://doi.org/10.1016/j.jhydrol.2025.133321. This paper also contains
a detailed description of model design and performance characteristics.

The seasonality component is based on the “three-sine” stream annual
temperature cycle function described in: Philippus, Corona, and Hogue,
2024, “Improved annual temperature cycle function for stream seasonal
thermal regimes”, *JAWRA*, https://doi.org/10.1111/1752-1688.13228.

Full training datasets, pre-trained models, and a knitted model
validation notebook PDF are available in CUAHSI HydroShare: `TempEst 2
Development Data: Observed Stream Temperature, Covariates, Performance
Data, and Analysis
Notebooks <https://www.hydroshare.org/resource/a8b243957f7946e388d10ab206990675/>`__.
