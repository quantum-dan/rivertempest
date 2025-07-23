libSCHEMA
=========

Generic Python implementation of the SCHEMA modeling framework with
built-in BMI/ngen support.

The SCHEMA (Seasonal Conditions Historical Expectation with Modeled
Anomaly) framework for hydrologic models, a generalization of stochastic
modeling, was introduced in TempEst 2 (Philippus et al. 2025) for the
TempEst family of ungauged stream temperature models, but it is
applicable to a wide range of models. This provides the opportunity to
(1) avoid a lot of boilerplate code and (2) provide a ready-to-go
`NextGen <https://www.weather.gov/media/owp/oh/docs/2021-OWP-NWM-NextGen-Framework.pdf>`__-compatible
`Basic Model
Interface <https://joss.theoj.org/papers/10.21105/joss.02317>`__ (BMI)
implementation, easing the development process for future models. Much
of the SCHEMA logic is totally model-agnostic, so that portion of the
code can be pre-written.

For that reason, this is a model-agnostic Python implementation of
SCHEMA, which specifies a general framework as well as providing
whatever functionality doesn’t depend on model specifics. All you have
to do for a specific implementation is define a few functions computing
seasonality and anomaly, as well as coefficient estimation if your
purpose is ungaged modeling.

Note: in Python, it is libschema (``import libschema``), not libSCHEMA.
Easier to type.

`Documentation Website <https://mines-ciroh.github.io/libSCHEMA/>`__

Quick Start
-----------

``pip install libschema``.

An API reference is included in ``api_ref.txt``, and the test
``tests/full_model.py`` serves as an example of a simple implementation.
Some key information:

-  The core SCHEMA implementation is ``libschema.SCHEMA`` (from
   ``model.py``). You can directly use the SCHEMA class to implement
   models, specifying seasonality and anomaly classes and the other
   required inputs. You can also extend it to add new functionality,
   such as automatically fitting a model from data. In the full-model
   test, the actual model implementation is two simple class definitions
   (26 lines of code) and the SCHEMA initialization. This provides a
   model that handles configuration files, stepwise or full-timeseries
   model execution, etc.
-  Templates for Anomaly, Seasonality, and ModEngine (coefficient
   modification engine) classes are in ``classes.py``.
-  The Basic Model Interface implementation for NextGen is
   ``libschema.SchemaBmi``, from ``bmi.py``. This should work out of the
   box, with the user just providing some metadata and a SCHEMA config
   file. I have not tested it yet, but it is derived from a known
   working implementation.
-  The ``analysis`` submodule provides some helper functions for
   cross-validation and goodness-of-fit metrics.

LibSCHEMA is free and open-source software and may be used, modified,
redistributed, etc so long as any software built upon it is also
open-source under the GNU General Public License v3. If you use
LibSCHEMA in your research, please cite Philippus et al. 2025.

General Concept
---------------

A SCHEMA model has three basic components: coefficient estimation,
seasonality, and anomaly. Coefficient estimation is too
application-specific for a generic implementation to be useful, so
that’s left to be handled externally. This implementation handles
seasonality (or any periodic component) and anomaly logic.

The basic approach is simple: compute the periodic component for the
timestep of interest, then compute the anomaly and add them together.
Why do we even need a library for that? With a simple implementation, we
do not, but the library can handle a lot of tricky legwork to provide
convenience features. Also, the library can provide a full-blown BMI
implementation out of the box that’s tested with NextGen, so that’s a
handy feature.

What sort of convenience features?

-  Smart “modification engines” that run periodically to adjust model
   components at runtime. These are great for handling things like
   climate change and drought, in the hydrologic use case.
-  Exporting coefficients to a data frame for analysis - super useful
   for the coefficient estimation part
-  Exporting models to, and reading them from, configuration files,
   which is a required capability for BMI/NextGen
-  Having the logic to run the model fast as a single series if there
   are no modification engines, or step-by-step if there are
-  And did I mention a BMI implementation?

More generally, it also separates concerns: the user just writes the
actual model mathematics without worrying about the implementation
logic. And this is huge, because practically any lumped model can be
implemented as SCHEMA with a little contortion. For instance, I’m fairly
sure you could just have no seasonal component and an LSTM for anomaly,
and you get a BMI-compatible LSTM for free. Or you could easily write a
unit hydrograph-based hydrologic model (seasonality = baseflow, anomaly
= unit hydrograph) with a couple of simple functions.

Basically, libSCHEMA minimizes the amount of software engineering you
have to worry about when building hydrologic models. The minimal example
model for testing is about 50 lines of code.

Using libSCHEMA also means that shared infrastructure can be developed
for libSCHEMA models in general, like the built-in BMI implementation.
In the future, that could be calibration utilities, parallelization
tools, or just about anything else - and it’ll work for any libSCHEMA
model.

Implemented Functionality
-------------------------

Current Status
~~~~~~~~~~~~~~

Core functionality has been implemented and tested with both a
full-fledged model implementation rewritten to use libSCHEMA. BMI
support is drafted, but has not been tested. Modification engines are
implemented in principle, but have not been tested.

Core Functionality
~~~~~~~~~~~~~~~~~~

-  Model initialization
-  Running the model
-  Exporting and importing model files
-  BMI, e.g. for NextGen

Convenience Features
~~~~~~~~~~~~~~~~~~~~

LibSCHEMA comes with a couple of utilities I built for my own use that
might be handy more broadly:

-  A suite of goodness-of-fit metrics (R2, RMSE, NSE, percent bias,
   max-miss/min-miss)
-  A flexible cross-validation function

Required Functionality
----------------------

The big pieces that needs to be implemented are the actual seasonality
and anomaly functions, which are provided as classes to make life easier
with configuration files, coefficient exporting, etc. Templates are
provided in ``classes.py``. Additionally, the implementation needs to
specify data requirements and the like. Seasonality and anomaly
functions should provide a vectorized implementation if you intend to
use ``run_series`` to run in a single pass.

If you want the model to be able to fit itself, you also need to define
a ``from_data`` method. Without that, a model can still be specified
with set coefficients, and you could calibrate it in the traditional
way, but it can’t automatically identify a fit.

If your implementation uses any modification engines, implementation for
those needs to be provided, and it may be useful to develop a custom
configuration-file retriever (otherwise libSCHEMA will try to pickle
them in the main config file).

Example
~~~~~~~

Here is a simple example with a toy model.

::

   class TestSeasonality(Seasonality):
       def __init__(self, mean, phase, amplitude, period):
           self.mean = mean
           self.phase = phase
           self.amplitude = amplitude
           self.period = period
       
       def apply(self, period):
           return self.mean + np.sin((period - self.phase)/self.period*6.28) * self.amplitude
       
       def apply_vec(self, period):
           return self.apply(period)
       

   class TestAnomaly(Anomaly):
       def __init__(self, sensitivity, window):
           self.sensitivity = sensitivity
           self.window = window
       
       def apply(self, periodic, period, anom_history):
           return self.sensitivity * np.mean(
               anom_history["x"][-self.window:])
       
       def apply_vec(self, periodic, period, anom_history):
           return self.sensitivity * anom_history["x"].rolling(
               self.window, min_periods=1).mean()
               
   sensitivity = 1
   window = 3
   mean = 5
   phase = 12
   amplitude = 8
   period = 36

   periodics = pd.DataFrame({
       "period": np.arange(36),
       "x": np.cos(np.arange(36))
       })

   model = SCHEMA(
       TestSeasonality(mean, phase, amplitude, period),
       TestAnomaly(sensitivity, window),
       periodics,
       [],  # no engines used
       ["T", "tp", "x"],  # input columns
       period,
       window
       )

   model.run_series(some_data, "T", 0, "tp")

Citation
--------

Philippus, Corona, Schneider, Rust, and Hogue, 2025, “Satellite-Based
Spatial-Statistical Modeling of Daily Stream Water Temperatures at the
CONUS Scale”, *Journal of Hydrology*,
doi:`10.1016/j.jhydrol.2025.133321 <https://doi.org/10.1016/j.jhydrol.2025.133321>`__.
