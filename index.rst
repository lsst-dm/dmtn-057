..
  Technote content.

  See https://developer.lsst.io/docs/rst_styleguide.html
  for a guide to reStructuredText writing.

  Do not put the title, authors or other metadata in this document;
  those are automatically added.

  Use the following syntax for sections:

  Sections
  ========

  and

  Subsections
  -----------

  and

  Subsubsections
  ^^^^^^^^^^^^^^

  To add images, add the image file (png, svg or jpeg preferred) to the
  _static/ directory. The reST syntax for adding the image is

  .. figure:: /_static/filename.ext
     :name: fig-label

     Caption text.

   Feel free to delete this instructional comment.

:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. Add content below. Do not include the document title.

.. note::

   **This technote is not yet published.**

Introduction
============

The Data Management team is beginning to use the ``lsst.verify`` framework to instrument the ``jointcal`` package and the Alert Production pipeline (through the ``ap_verify`` package).
As the first users, ``jointcal`` and ``ap_verify`` will play a leading role in establishing how ``verify`` is used by Stack packages.
Since we expect, in the long run, that the ``verify`` framework will be used throughout the DM Stack and by multiple pipelines, it is essential that we develop a system that allows new verification metrics to be supported by pipeline tasks, and allows new tasks to easily adopt existing metrics, without introducing undue development overhead or technical debt.

This note presents several design sketches to study the feasibility of different ways to use the ``verify`` system within tasks.
It reviews how ``ap_verify`` and ``jointcal`` expect to be able to verify their pipelines, as well as the existing capabilities of the ``verify`` framework, before presenting a series of proposed designs for how to best integrate the existing framework into arbitrary tasks.
Each subsection of :ref:`Proposed Stack Architectures <arch>` describes the expected costs and benefits of one approach, along with archetypal examples to illustrate the implications for metric or task implementations.
The intent is that one of the proposals can serve as the basis for a later, more formal, design and subsequent implementation in the Stack.

.. _design-goals:

Design Goals
============

At present, there is no centralized record of what either ``ap_verify`` or ``jointcal`` expect from the ``verify`` framework.
The following list has been put together from internal conversations within the Alert Production project, discussions with the SQuaSH group, and personal knowledge of the state of the DM Stack.

I assume the designs presented in this tech note must have or enable the following capabilities:

#. Developers must be able to define, measure, and manage a large number of metrics (likely thousands).
#. Developers must be able to define metrics that apply to any task, metrics that apply to only certain categories of tasks, metrics that are specific to a particular task implementation, and metrics that combine information from multiple tasks.
#. In the former two cases, it must be possible to track measurements on subtasks according to both the task implementation class and the subtask's role (e.g., metrics analysis must be able to distinguish running time of ``AstrometryTask`` from running time of ``imageDifference.astrometer``).
#. It must be easy to define and adopt new metrics as construction and commissioning proceed.
#. It must be easy to add and instrument new tasks as construction and commissioning proceed.
#. It must continue to be possible to run a task without forcing the user to perform metrics-related setup or use metrics-related features.
#. Measurements must be produced with provenance information or other context that might help interpret them.
#. Metrics must be definable in a way that makes no assumptions about how or whether a dataset is divided among multiple task invocations for processing (e.g., for parallel computing).
   Following `DMTN-055`_, this division shall be called "units of work" in the following discussion.
#. It must be possible to compute the same quantity (e.g., running time, source counts, image statistics) at different stages of the pipeline; for different visits, CCDs, patches, or sources; or for different photometric bands.
#. It must be easy to distinguish such multiple measurements (e.g., request image statistics after ISR or after background subtraction) when analyzing metrics or their trends.
#. If a metric is expensive to calculate, it must be possible to control whether it is computed for a particular run or in a particular environment.
   It must be possible to do so without disabling simple metrics that would be valuable in operations.
#. It should be easy to integrate verification into the SuperTask framework (as currently planned, or with probable design changes), but it must also be possible to implement it entirely in the current Stack.

.. _use-cases:

The following use cases will be used to illustrate the consequences of each design for metric implementations:

Running Time
    Measures the wall-clock time, in seconds, needed to execute a particular task.
    Should report the total running time for multiple units of work.
    Applicable to all tasks.
Number of Sources Used for Astrometric Solution
    Measures the number of points used to constrain an image's astrometric fit.
    Should report the total number used for multiple units of work.
    Applicable to ``AstrometryTask`` and any substitutes.
Fraction of Science Sources that Are DIASources
    Measures the rate at which DIASources appear in test data.
    Should report the weighted average for multiple units of work.
    Requires combining information from ``CalibrateTask`` and ``ImageDifferenceTask``, or their substitutes.
DCR Goodness of Fit
    Measures the quality of the model used to correct for Differential Chromatic Refraction.
    The goodness-of-fit statistic used may be configurable, and which definition was adopted must be reported as measurement metadata.
    Should report a statistically rigorous combination for multiple units of work.
    Applicable to ``DcrMatchTemplateTask``; ignored if DCR is not used for a particular pipeline run.


The lsst.verify Framework
=========================

The ``verify`` framework and its usage are presented in detail in `SQR-019`_.
This section only discusses the aspects that will constrain designs for using ``verify`` in tasks.

``Metric`` objects represent the abstract quantity to be measured.
They include such metadata as the quantity's units and a description.
Metrics can be defined by any package, but by convention should be centralized in the ``verify_metrics`` package.
Most methods in ``verify`` assume by default that all metrics are defined in ``verify_metrics``.
Metrics are globally identified by name, with a namespace corresponding to the package that will measure the metric.
For example, ``verify_metrics`` can define a metric named ``"ip_diffim.dia_source_per_direct_source"``, which can be looked up by any code that uses ``verify`` classes.

``Measurement`` objects represent the value of a ``Metric`` for a particular run.
They must be defined using an Astropy ``Quantity`` and the name of the metric being measured, and may contain supplementary data (including vector data) or arbitrary metadata.
A ``Measurement`` can be passed as part of a task's return value, or persisted using a ``Job`` object, but cannot be stored as task metadata.

A ``Job`` manages collections of ``Measurements``, including persistence, local analysis, and SQuaSH submission.
A single ``Job`` must contain at most one measurement of a particular metric.
``Jobs`` can also contain metadata, such as information about the runtime environment.
``Jobs`` can be merged (it is not clear how conflicting measurements or metadata are handled), and can be persisted to or unpersisted from user-specified files.
``Job`` persistence will be replaced with Butler-based persistence at a later date.

.. _arch:

Proposed Stack Architectures
============================

This section presents four options for how to incorporate ``verify`` measurements into the tasks framework.
The first is to :ref:`handle as much information as possible within task metadata<arch-metadata>`, as originally envisioned for the task framework.
The second is to :ref:`have tasks create Measurements in-place<arch-direct>`, as originally envisioned for the verify framework.
The third is to :ref:`delegate Measurement creation to autonomous factory objects<arch-observer>` that can be freely added or removed from a task.
The fourth is to :ref:`delegate Measurement creation to high-level factory objects<arch-visitor>` that inspect the hierarchy of tasks and subtasks.

.. _arch-metadata:

Pass Metadata to a Central Measurement Package
----------------------------------------------

.. _arch-metadata-structure:

Architecture and Standard Components
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

In this design, all information of interest to metrics will be stored in a task's metadata.
The metadata will be passed up to a dedicated package (named, for example, ``verify_measurements``) that will find the appropriate keys and create ``Measurement`` objects.
High-level handling of the measurements can be done by a single ``Job`` object.
A prototype of this approach is used in the ``lsst.ap.verify.measurements`` package to handle running times, but in the interests of portability to other pipelines the final code should be a dependency of ``ap_verify`` rather than a subpackage.

To minimize coupling with the task itself, the code that performs the measurements can be placed in decorators analogous to ``lsst.pipe.base.timeMethod``.
This approach also avoids code duplication for metrics that apply to more than one task class.
However, as the number of metrics grows, so will the number of decorators attached to a class's ``run`` method.
Related metrics can be grouped in one decorator; for example, ``timeMethod`` measures not only timing, but also memory usage and other forms of profiling.

While tasks or their decorators are necessarily coupled to ``verify_metrics``, ``verify_measurements`` need not know about most defined metrics if the metadata keys follow a particular format that allows discovery of measurements by iterating over the metadata (e.g., ``"<task-prefix>.verify.measurements.foo"`` for a metric named ``"package.foo"``).
Since the correct way to merge measurements from multiple units of work depends on the metric (for example, the four use cases described :ref:`above <use-cases>` require three different approaches), a standardized key (perhaps ``"<task-prefix>.verify.combiners.foo"``) can be used to specify the algorithm to combine the data.
The use of a string to indicate the combiner only scales well if the majority of metrics share a small number of combiners, such as sum or average.

.. figure:: /_static/metadata_data_flow.svg
   :name: fig-metadata-sequence
   :target: _static/metadata_data_flow.svg

   Illustration of how measurement data are passed up from tasks in the metadata-based architecture.
   ``anInstance`` and ``anotherInstance`` are ``ConcreteCmdLineTask`` objects run on different data.

Standardized metadata keys cannot handle metrics that depend on the results of multiple tasks (such as the :ref:`DIASource fraction<arch-metadata-examples-fdia>`).
In this case, information can still be passed up through metadata, but tasks should *avoid* using the ``verify.measurement`` prefix so that generic ``Measurement``-making code does not mistakenly process them.
Instead, each cross-task metric will need its own function in ``verify_measurements`` to search across all task classes for the relevant information and make a ``Measurement``.
Handling of cross-task metrics must therefore be coordinated across at least three packages -- ``verify_metrics``, the task package(s), and ``verify_measurements``.

Standardized metadata keys can be used to record supplementary information about a measurement, for example by using ``verify.extras`` and ``verify.notes`` PropertySets.

.. _arch-metadata-workload:

Requirements for Task Creators and Maintainers
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The main requirement imposed on authors of new tasks is the use of measurement decorators.
It may be necessary to ensure decorators are applied in a particular order (for example, ``timeMethod`` should not include measurement overhead, so it should be listed last).
If the decorators make assumptions about a task's fields, they may constrain the implementation of the task itself.
Implementation constraints go away if measurement metadata are written directly by a task's methods, but then the task author is responsible for following all the conventions described :ref:`above<arch-metadata-structure>`, including specifying a combiner and any other auxiliary metadata keys.

Custom task runners that call ``run`` multiple times per ``Task`` object must copy the object's metadata after each run, to keep it from getting lost.
(This is not a problem for ``TaskRunner``, which creates a new ``Task`` for each run.)

If all verification-related work is done by decorators, than maintaining instrumented tasks is easy; ``Task`` code can be changed and decorators added or removed as desired.
The only risk is if decorators constrain task implementations in some way; such details must be clearly marked as unchangeable.
If decorators depend on particular metadata keys being available, the lines that write those keys must be kept in sync with the key names passed to decorators (see :ref:`DCR goodness of fit<arch-metadata-examples-dcrgof>`).
If tasks write measurement metadata directly, then maintainers must know not to touch those lines in any way.

Authors of new metrics must implement a decorator that measures them, most likely in ``pipe_base`` or a specific task's package, and add it to all relevant task classes.
The decorator must conform to all conventions regarding metadata keys.
If the metric requires a new way to combine units of work, the new combiner must be implemented and registered under a unique name in ``verify_measurements``.

.. _arch-metadata-procon:

Advantages and Disadvantages
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

A metadata-driven architecture limits changes to the task framework to imposing a convention for metadata keys; tasks need not depend on ``verify`` at all.
However, it does require a centralized ``Measurement``-making package that frameworks like ``ap_verify`` or ``validate_drp`` must call after all tasks have been run.

Adding most metrics requires changes to two packages (the minimum allowed by the ``verify`` framework), but cross-task metrics require three.
Metrics cannot be added to or removed from a task without modifying code.
Configs could be used to disable them, although this breaks the separation of task and instrumentation code somewhat.

Dividing a dataset into multiple units of work is poorly supported by a metadata-based architecture, because each metric may require a different way to synthesize a full-dataset measurement from the individual measurements, yet metadata does not allow code to be attached to measurements.
On the other hand, it is very easy to support tracking of subtask measurements by both class and role, because the metadata naturally provide by-role information.

The biggest weakness of this architecture may well be its dependence on convention: metadata keys that don't conform to the expected format must, in many cases, be silently ignored.

.. _arch-metadata-examples:

Example Metric Implementations
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Note: in practice, all the metadata keys seen by ``verify_measurements`` would be prefixed by the chain of subtasks that produced them, requiring more complex handling than a lookup by a fixed name.
This extra complexity is ignored in the examples, but is fairly easy to implement.

.. _arch-metadata-examples-time:

Running Time
""""""""""""

This measurement can be implemented by modifying the existing ``timeMethod`` decorator to use a standardized metric name in addition to the existing keys.
The new key would need to take the difference between start and end times instead of storing both:

.. code-block:: py

   obj.metadata.add(name = "verify.measurements.%s_RunTime" % className,
                    value = deltaT)
   obj.metadata.add(name = "verify.combiners.%s_RunTime" % className,
                    value = "sum")

This example assumes that each task needs a unique metric to represent its running time, as is the case with the current ``verify`` framework.
If a later version allows a single running time metric to be measured by each task, then the metric name need no longer contain the class name.

.. _arch-metadata-examples-nastro:

Number of Sources Used for Astrometric Solution
"""""""""""""""""""""""""""""""""""""""""""""""

Astrometric tasks already report the number of sources used in the fitting process, so the decorator can be a simple wrapper:

.. code-block:: py
   :emphasize-lines: 1-12,16,23

   def numAstroSources():
       @wraps(func)
       def wrapper(self, *args, **kwargs):
           result = func(self, *args, **kwargs)
           # Any substitute for AstrometryTask must share its return value spec
           nSources = len(result.matches)
           self.metadata.add(name = "verify.measurements.NumAstroSources",
                            value = nSources)
           self.metadata.add(name = "verify.combiners.NumAstroSources",
                            value = "sum")
           return result
       return wrapper

   class AstrometryTask(RefMatchTask):
       ...
       @numAstroSources
       @pipeBase.timeMethod
       def run(self, sourceCat, exposure):
           ...

   class BetterAstrometryTask(RefMatchTask):
       ...
       @numAstroSources
       @pipeBase.timeMethod
       def run(self, sourceCat, exposure):
           ...

.. _arch-metadata-examples-fdia:

Fraction of Science Sources that Are DIASources
"""""""""""""""""""""""""""""""""""""""""""""""

This metric requires combining information from ``CalibrateTask`` and ``ImageDifferenceTask``.
This approach requires one decorator each to store the numerator and denominator, and some custom code to compute the fraction:

.. code-block:: py
   :emphasize-lines: 1-9,13,19-27,31

   def numScienceSources():
       @wraps(func)
       def wrapper(self, *args, **kwargs):
           result = func(self, *args, **kwargs)
           nSources = len(result.sourceCat)
           self.metadata.add(name = "verify.fragments.NumScienceSources",
                            value = nSources)
           return result
       return wrapper

   class CalibrateTask(RefMatchTask):
       ...
       @numScienceSources
       @pipeBase.timeMethod
       def run(self, dataRef, exposure=None, background=None, icSourceCat=None,
           doUnpersist=True):
           ...

   def numDiaSources():
       @wraps(func)
       def wrapper(self, *args, **kwargs):
           result = func(self, *args, **kwargs)
           nSources = len(result.sources)
           self.metadata.add(name = "verify.fragments.NumDiaSources",
                            value = nSources)
           return result
       return wrapper

   class ImageDifferenceTask(RefMatchTask):
       ...
       @numDiaSources
       @pipeBase.timeMethod
       def run(self, sensorRef, templateIdList=None):
           ...

And, in ``verify_measurements``,

.. code-block:: py
   :emphasize-lines: 1-17,21-23

   def measureDiaSourceFraction(allVerifyMetadata):
       SCIENCE_KEY = "fragments.NumScienceSources"
       DIA_KEY = "fragments.NumDiaSources"
       scienceSources = 0
       diaSources = 0
       for oneRunMetadata in allVerifyMetadata:
           if oneRunMetadata.exists(SCIENCE_KEY):
               scienceSources += oneRunMetadata.get(SCIENCE_KEY)
           if oneRunMetadata.exists(DIA_KEY):
               diaSources += oneRunMetadata.get(DIA_KEY)

       # Generic Measurements are not created if code not run, be consistent
       if scienceSources > 0:
           return lsst.verify.Measurement(
               "Fraction_DiaSource_ScienceSource",
               (diaSources / scienceSources) * u.dimensionless_unscaled))
       else:
           return None

   def makeSpecializedMeasurements(allVerifyMetadata):
       ...
       measurement = measureDiaSourceFraction(allVerifyMetadata)
       if measurement is not None:
           job.measurements.insert(measurement)
       ...

Note that ``measureDiaSourceFraction`` naturally takes care of the problem of combining measurements from multiple units of work.

.. _arch-metadata-examples-dcrgof:

DCR Goodness of Fit
"""""""""""""""""""

``DcrMatchTemplateTask`` does not yet exist, but I assume it would report goodness-of-fit in the task metadata even in the absence of a verification framework.
The main complication is that there may be different ways to compute goodness of fit, and each statistic may require its own combiner, so this information must be provided along with the measurement.

.. code-block:: py
   :emphasize-lines: 1-19,23

   def dcrGoodnessOfFit(valueKey, typeKey):
       def customWrapper(func):
           @wraps(func)
           def wrapper(self, *args, **kwargs):
               try:
                   return func(self, *args, **kwargs)
               finally:
                   if self.metadata.exists(valueKey) and self.metadata.exists(typeKey):
                       gofValue = self.metadata.get(valueKey)
                       gofType = self.metadata.get(typeKey)
                       self.metadata.add(name = "verify.measurements.DcrGof",
                                        value = gofValue)
                       self.metadata.add(name = "verify.combiners.DcrGof",
                                        value = "dcrStatCombine")
                       # added to Measurement's `notes` member, AND needed by dcrStatCombine
                       self.metadata.add(name = "verify.notes.DcrGof.gofStatistic",
                                        value = gofType)
           return wrapper
       return customWrapper

   class DcrMatchTemplateTask(CmdLineTask):
       ...
       @dcrGoodnessOfFit("gof", "gofType")
       @pipeBase.timeMethod
       def run(self, dataRef, selectDataList=[]):
           ...

One could avoid duplicating information between ``gof`` and ``verify.measurements.DcrGof`` by having ``DcrMatchTemplateTask`` write the ``verify.*`` keys directly from ``run`` instead of using a decorator.
However, mixing a task's primary and verification-specific code in this way could make it harder to understand and maintain the code, and recording metadata only in a verification-compatible format would make it hard to use by other clients.

Regardless of how the keys are written, ``verify_measurements`` would need a custom combiner:

.. code-block:: py

   def dcrStatCombine(allVerifyDcrMetadata):
       try:
           statisticType = allVerifyDcrMetadata[0].get(
               "notes.DcrGof.gofStatistic")
           if statisticType == "Chi-Squared":
               chisqCombine(allVerifyDcrMetadata)
           elif ...

.. _arch-direct:

Make Measurements Directly
--------------------------

.. _arch-direct-structure:

Architecture and Standard Components
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

In this design, ``Measurement`` objects will be made by tasks.
Tasks will have a ``Job`` object (``Task.job``) for collecting their ``Measurements``, which can be either persisted or passed upward as part of a task's return value.
High-level handling of all ``Measurements`` would be handled by a ``Job`` living in a verification package (such as ``ap_verify``), which consolidates the task-specific ``Job`` objects.

To minimize coupling with the task itself, the code that creates the ``Measurements`` can be placed in decorators similar to ``lsst.pipe.base.timeMethod``, except that the decorators would update ``Task.job`` rather than ``Task.metadata``.
This approach also avoids code duplication for metrics that apply to more than one task class.
However, as the number of metrics grows, so will the number of decorators attached to a class's ``run`` method.
Related metrics can be grouped in one decorator; for example, ``timeMethod`` measures not only timing, but also memory usage and other forms of profiling.

Measurements may depend on information that is internal to ``run`` or a task's other methods.
If this is the case, the ``Measurement`` may be created by an ordinary function called from within ``run``, instead of by a decorator, or the internal information may be stored in metadata and then extracted by the decorator.

Directly constructed ``Measurements`` cannot handle metrics that depend on the results of multiple tasks (such as the :ref:`DIASource fraction<arch-direct-examples-fdia>`); such metrics must be measured in a centralized location.
There are two ways to handle cross-task measurements:

#. The necessary information can be stored in :ref:`metadata<arch-metadata>`, and computed as a special step after all tasks have been run.
#. We can impose a requirement that all cross-task metrics be expressible in terms of single-task metrics.
   In the DIASource fraction example such a requirement is a small burden, since both "Number of detected sources" and "Number of DIASources" are interesting metrics in their own right, but this may not be the case in general.

The correct way to merge measurements from multiple units of work depends on the metric (for example, the four use cases described :ref:`above <use-cases>` require three different approaches).
This information can be provided by requiring that ``Measurement`` objects include a merging function, which can be invoked either as part of the task parallelization framework (as shown in the :ref:`figure<fig-direct-sequence>`), or as part of a measurement-handling package (as required by the :ref:`metadata-based architecture<arch-metadata-structure>`).

.. figure:: /_static/direct_data_flow.svg
   :name: fig-direct-sequence
   :target: _static/direct_data_flow.svg

   Illustration of how measurements are handled in the direct-measurement and observer-based architectures, assuming ``Job`` persistance is not used and multiple units of work are combined as part of the existing parallelism framework.
   ``anInstance`` and ``anotherInstance`` are ``ConcreteCmdLineTask`` objects run on different data.
   The subtask of ``anotherInstance`` and the ``Measurement`` it produces are omitted for clarity.

.. _arch-direct-workload:

Requirements for Task Creators and Maintainers
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The main requirement imposed on authors of new tasks is the use of measurement decorators or functions.
It may be necessary to ensure measurements are made in a particular order (for example, timing should not include measurement overhead).
If measurement decorators make assumptions about a task's fields, they may constrain the implementation of the task itself.
Functions called from within ``run`` do not impose implementation constraints, but may be less visible to maintainers if they are buried in the rest of the task code.

If ``verify`` does not support multiple measurements of the same metric, then any task runner that calls ``run`` multiple times per ``Task`` object must extract the object's job after each run, to prevent information from being lost.
(This is not a problem for ``TaskRunner``, which creates a new ``Task`` object for each run.)

If all verification-related work is done by decorators, than maintaining instrumented tasks is easy; task code can be changed and decorators added or removed as desired.
The only major risk is if decorators constrain task implementations in some way; such details must be clearly marked as unchangeable.
If measurements are made by functions called from within ``run``, then the maintainability of the task depends on how well organized the code is -- if measurement-related calls are segregated into their own block, maintainers can easily work around them.

Authors of new metrics must implement a decorator or function that measures them, most likely in ``pipe_base`` or a specific task's package, and add it to all relevant task classes.
The decorator or function must ensure the resulting ``Measurement`` has a combining functor.
Standard combiners may be made available through a support package to reduce code duplication.

.. _arch-direct-procon:

Advantages and Disadvantages
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

A direct-measurement architecture minimizes changes needed to the ``verify`` framework, which already assumes each task is responsible for persisting Job information.

Adding most metrics requires changes to two packages (the minimum allowed by the ``verify`` framework), but cross-task metrics require either two (if all single-task components are themselves metrics) or three (if the implementation is kept as general as possible).
Metrics cannot be added to or removed from a task without modifying code.
Configs could be used to disable them, although this breaks the separation of task and instrumentation code somewhat.

Because of its decentralization, a direct-measurement architecture has trouble supporting cross-task metrics; in effect, one needs one framework for single-task metrics and another for cross-task metrics.

.. _arch-direct-examples:

Example Metric Implementations
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. _arch-direct-examples-time:

Running Time
""""""""""""

The existing ``timeMethod`` decorator handles finding the running time itself, so the ``Measurement``-making decorator only needs to package the information.
Since this design imposes a dependency between two decorators, the new decorator raises an exception if the ``timeMethod`` decorator is not used.

.. code-block:: py
   :emphasize-lines: 1-19,23

   def timeMeasurement():
       @wraps(func)
       def wrapper(self, *args, **kwargs):
           try:
               return func(self, *args, **kwargs)
           finally:
               try:
                   start = self.metadata.get("runStartCpuTime")
                   end = self.metadata.get("runEndCpuTime")
               except pexExceptions.NotFoundError as e:
                   raise AttributeError(
                       "@timeMethod must be listed after @timeMeasurement"
                   ) from e
               metricName = "%s_RunTime" % type(self).__name__
               measurement = lsst.verify.Measurement(metricName,
                                                     (end - start) * u.seconds))
               measurement.combiner = verify.measSum
               self.job.measurements.insert(measurement)
       return wrapper

   class AFancyTask(Task):
       ...
       @timeMeasurement
       @pipeBase.timeMethod
       def run(self, data):
           ...

This example assumes that each task needs a unique metric to represent its running time, as is the case with the current ``verify`` framework.
If a later version allows a single running time metric to be measured by each task, then the metric name need no longer contain the class name.

.. _arch-direct-examples-nastro:

Number of Sources Used for Astrometric Solution
"""""""""""""""""""""""""""""""""""""""""""""""

Astrometric tasks already report the number of sources used in the fitting process, so the decorator can be a simple wrapper:

.. code-block:: py
   :emphasize-lines: 1-13,17,24

   def numAstroSources():
       @wraps(func)
       def wrapper(self, *args, **kwargs):
           result = func(self, *args, **kwargs)
           # Any substitute for AstrometryTask must share its return value spec
           nSources = len(result.matches)
           measurement = lsst.verify.Measurement(
               "NumAstroSources",
               nSources * u.dimensionless_unscaled))
           measurement.combiner = verify.measSum
           self.job.measurements.insert(measurement)
           return result
       return wrapper

   class AstrometryTask(RefMatchTask):
       ...
       @numAstroSources
       @pipeBase.timeMethod
       def run(self, sourceCat, exposure):
           ...

   class BetterAstrometryTask(RefMatchTask):
       ...
       @numAstroSources
       @pipeBase.timeMethod
       def run(self, sourceCat, exposure):
           ...

.. _arch-direct-examples-fdia:

Fraction of Science Sources that Are DIASources
"""""""""""""""""""""""""""""""""""""""""""""""

This metric requires combining information from ``CalibrateTask`` and ``ImageDifferenceTask``.

The source counts can be passed to verification code using an approach similar to that given for the :ref:`metadata-based architecture<arch-metadata-examples-fdia>`.
The only difference is that the location of ``makeSpecializedMeasurements`` depends on whether Jobs are handled directly by ``CmdLineTask``, or in a higher-level package using persistence.

If instead the framework requires that the number of science sources and number of DIASources be metrics, one implementation would be:

.. code-block:: py
   :emphasize-lines: 1-12,16,22-33,37

   def numScienceSources():
       @wraps(func)
       def wrapper(self, *args, **kwargs):
           result = func(self, *args, **kwargs)
           nSources = len(result.sourceCat)
           measurement = lsst.verify.Measurement(
               "NumScienceSources",
               nSources * u.dimensionless_unscaled))
           measurement.combiner = verify.measSum
           self.job.measurements.insert(measurement)
           return result
       return wrapper

   class CalibrateTask(RefMatchTask):
       ...
       @numScienceSources
       @pipeBase.timeMethod
       def run(self, dataRef, exposure=None, background=None, icSourceCat=None,
           doUnpersist=True):
           ...

   def numDiaSources():
       @wraps(func)
       def wrapper(self, *args, **kwargs):
           result = func(self, *args, **kwargs)
           nSources = len(result.sources)
           measurement = lsst.verify.Measurement(
               "NumDiaSources",
               nSources * u.dimensionless_unscaled))
           measurement.combiner = verify.measSum
           self.job.measurements.insert(measurement)
           return result
       return wrapper

   class ImageDifferenceTask(RefMatchTask):
       ...
       @numDiaSources
       @pipeBase.timeMethod
       def run(self, sensorRef, templateIdList=None):
           ...

.. code-block:: py
   :emphasize-lines: 1-12,16-19

   def measureFraction(job, metric, numeratorName, denominatorName):
       try:
           numerator = job.measurements[numeratorName]
           denominator = job.measurements[denominatorName]
       except KeyError:
           # Measurements not made, fraction not applicable
           return

       fraction = numerator.quantity / denominator.quantity
       measurement = lsst.verify.Measurement(metric, fraction)
       # TODO: how to handle extras and notes?
       job.measurements.insert(measurement)

   def makeSupplementaryMeasurements(masterJob):
       ...
       measureFraction(masterJob,
                       "Fraction_DiaSource_ScienceSource",
                       "NumDiaSources",
                       "NumScienceSources")
       ...

Unlike the solution given in the :ref:`metadata-based architecture<arch-metadata-examples-fdia>`, this implementation assumes that merging of multiple units of work is handled by ``NumDiaSources`` and ``NumScienceSources``.

.. _arch-direct-examples-dcrgof:

DCR Goodness of Fit
"""""""""""""""""""

``DcrMatchTemplateTask`` does not yet exist, but I assume it would report goodness-of-fit in the task metadata even in the absence of a verification framework.
The decorator wraps the metadata in a ``Measurement``.

.. code-block:: py
   :emphasize-lines: 1-3, 5-22,26

   def chisqCombine(measurements):
       """Compute a chi-squared Measurement for a data set from values for subsets."""
       ...

   def dcrGoodnessOfFit(valueKey, typeKey):
       def customWrapper(func):
           @wraps(func)
           def wrapper(self, *args, **kwargs):
               try:
                   return func(self, *args, **kwargs)
               finally:
                   if self.metadata.exists(valueKey) and self.metadata.exists(typeKey):
                       gofValue = self.metadata.get(valueKey)
                       gofType = self.metadata.get(typeKey)
                       measurement = lsst.verify.Measurement(
                           "DcrGof",
                           gofValue * getUnits(gofType))
                       measurement.combiner = getCombiner(gofType)
                       measurement.notes['gofStatistic', gofType]
                       self.job.measurements.insert(measurement)
           return wrapper
       return customWrapper

   class DcrMatchTemplateTask(CmdLineTask):
       ...
       @dcrGoodnessOfFit("gof", "gofType")
       @pipeBase.timeMethod
       def run(self, dataRef, selectDataList=[]):
           ...

.. _arch-observer:

Use Observers to Make Measurements
----------------------------------

.. _arch-observer-structure:

Architecture and Standard Components
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

In this design, ``Measurement`` objects will be made by factory objects separate from the task itself.
Tasks will have a ``Job`` object for collecting their measurements, which can be either persisted or passed upward as part of a task's return value.
High-level handling of all measurements can be handled by a ``Job`` living in a verification package (such as ``ap_verify``), which consolidates the task-specific ``Job`` objects.

The factories for the appropriate metrics will be registered with a task at construction time, using a new method (called ``Task.addListener``, to allow for future applications other than metrics).
The registration can be made configurable, although if each metric has its own factory, the config file will be an extra place that must be kept in sync with metrics definitions in ``verify_metrics``.
If one class measures multiple related metrics, then config changes are needed less often.

A task has a method (``Task.notify``) that triggers its registered factories on one of several standardized events (the :ref:`examples <arch-observer-examples>` assume there are three: Begin, Abort, and Finish); the events applicable to a given factory are specified at registration.
Factories query the task's metadata for information they need, make the appropriate ``Measurement`` object(s), and pass them back to the task's ``Job``.

Measurements may depend on information that is internal to ``run`` or a task's other methods.
If this is the case, internal information may be stored in metadata and then extracted by the factory.

If metrics depend on the results of multiple tasks (such as the :ref:`DIASource fraction<arch-observer-examples-fdia>`), they can be worked around using the same techniques as for :ref:`direct measurements<arch-direct-structure>`.
It is also possible to handle cross-task metrics by registering the same factory object with two tasks.
However, supporting such a capability would require that factories be created and attached to tasks from above, which would take away this framework's chief advantage -- that it does not require centralized coordination, but is instead largely self-operating.
See the :ref:`visitor pattern<arch-visitor-structure>` for a design that does handle cross-task metrics this way.

.. figure:: /_static/observer_data_flow.svg
   :name: fig-observer-sequence
   :target: _static/observer_data_flow.svg

   Illustration of how measurements are created in the observer-based architecture, assuming all measurement information is available through ``metadata``.
   Handling of measurements once they have been created works the same as for the :ref:`direct measurement architecture<fig-direct-sequence>`.

The correct way to merge measurements from multiple units of work depends on the metric (for example, the four use cases described :ref:`above <use-cases>` require three different approaches).
This information can be provided by requiring that ``Measurement`` objects include a merging function.

.. _arch-observer-workload:

Requirements for Task Creators and Maintainers
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Authors of new tasks must include in the task configuration information indicating which factories are to be attached to a task.
The convention for defaults may be to register either all applicable factories, or a subset that is deemed to have little runtime overhead.
The registration process itself can be handled by ``Task.__init__`` with no direct developer intervention.

If ``verify`` does not support multiple measurements of the same metric, then any task runner that calls ``run`` multiple times per ``Task`` object must extract the object's job after each run, to prevent information from being lost.
(This is not a problem for ``TaskRunner``, which creates a new ``Task`` object for each run.)

In general, maintaining instrumented tasks is easy.
The only risk is if factories constrain task implementations in some way; such details must be clearly marked as unchangeable.
If factories depend on particular metadata keys being available, the lines that write those keys must be kept in sync with the key names assumed by factories.

Authors of new metrics must implement a factory that measures them, most likely in ``pipe_base`` or a specific task's package, and add it to all relevant configs.
The factory must ensure the resulting ``Measurement`` has a combining functor, as for direct construction of ``Measurements``.

.. _arch-observer-procon:

Advantages and Disadvantages
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

An observer-based architecture provides maximum decentralization of responsibility: not only is each task responsible for handling its own measurements, but little to no task code needs to be aware of the specific metrics defined for each task.
While the observer architecture is not the only one that allows run-time configuration of metrics, it is the one where such configuration fits most naturally by far.
However, the high decentralization also gives it the worst support for cross-task metrics.

Adding single-task metrics requires changes to two packages, the minimum allowed by the ``verify`` framework.
Metrics can be enabled and disabled at will.

Extracting measurements from a task may require that a task write metadata it normally would not, duplicating information and forcing a task to have some knowledge of its metrics despite the lack of explicit references in the code.

It would be difficult to retrofit ``notify`` calls into the existing tasks framework.
If task implementors are responsible for calling ``notify`` correctly, the requirement is difficult to enforce.
If ``Task`` is responsible, then tasks would need one ``run`` method that serves as the API point of entry (for example, for use by ``TaskRunner``), and a second workhorse method to be implemented by subclasses.
Either approach involves significant changes to existing code.

.. _arch-observer-examples:

Example Metric Implementations
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

These examples assume that ``InvalidMeasurementError`` is handled by ``notify`` to prevent metrics-related errors from leaking into primary task code.

.. _arch-observer-examples-time:

Running Time
""""""""""""

In this design, it would be easier for the factory to perform the timing itself than to copy the measurements from ``timeMethod`` (or any other decorator on ``run``).
Note that there is no way to guarantee that the running time factory handles Finish before any other measurement factories do.

.. code-block:: py

   class RunningTimeMeasurer:
       def __init__(self, task):
           self.task = task

       def update(event):
           if (event == "Begin"):
               self._start = time.clock()
           elif (event == "Abort" || event == "Finish"):
               try:
                   deltaT = time.clock() - self._start
               catch AttributeError as e:
                   raise InvalidMeasurementError("No Begin event detected") from e
               metricName = "%s_RunTime" % type(self.task).__name__
               measurement = lsst.verify.Measurement(metricName,
                                                     deltaT * u.seconds))
               measurement.combiner = verify.measSum
               self.task.job.measurements.insert(measurement)

Assuming users don't just adopt the default settings, the config file for a task might look something like:

.. code-block:: py

   config.listeners['RunningTimeMeasurer'] = EventListenerConfig()
   config.listeners['RunningTimeMeasurer'].events = ['Begin', 'Abort', 'Finish']

.. _arch-observer-examples-nastro:

Number of Sources Used for Astrometric Solution
"""""""""""""""""""""""""""""""""""""""""""""""

Astrometric tasks report the number of sources used in the fitting process, but this information is not easily available at update time.
This implementation assumes all returned information is also stored in metadata.

This implementation also assumes that the config system allows constructor arguments to be specified, to minimize code duplication.

.. code-block:: py

   class SourceCounter:
       def __init__(self, task, metric):
           self.task = task
           self.metricName = metric

       def update(event):
           if (event == "Finish"):
               try:
                   nSources = self.metadata.get('sources')
               except KeyError as e:
                   raise InvalidMeasurementError(
                       "Expected `sources` metadata keyword"
                       ) from e
               measurement = lsst.verify.Measurement(
                   self.metricName,
                   nSources * u.dimensionless_unscaled))
               measurement.combiner = verify.measSum
               self.task.job.measurements.insert(measurement)

Assuming users don't just adopt the default settings, the config file might look something like:

.. code-block:: py

   astrometer.listeners['SourceCounter'] = EventListenerConfig()
   astrometer.listeners['SourceCounter'].args = ['NumAstroSources']  # Metric name
   astrometer.listeners['SourceCounter'].events = ['Finish']

.. _arch-observer-examples-fdia:

Fraction of Science Sources that Are DIASources
"""""""""""""""""""""""""""""""""""""""""""""""

This metric requires combining information from ``CalibrateTask`` and ``ImageDifferenceTask``.
The source counts can be passed to verification code using an approach similar to that given for the :ref:`metadata-based architecture<arch-metadata-examples-fdia>`.
The only difference is that the location of ``makeSpecializedMeasurements`` depends on whether Jobs are handled directly by ``CmdLineTask``, or in a higher-level package using persistence.

.. _arch-observer-examples-dcrgof:

DCR Goodness of Fit
"""""""""""""""""""

``DcrMatchTemplateTask`` does not yet exist, but I assume it would report goodness-of-fit in the task metadata even in the absence of a verification framework.
The factory wraps the metadata in a ``Measurement``.

.. code-block:: py

   class DcrGoodnessOfFitMeasurer:
       def __init__(self, task):
           self.task = task

       def update(event):
           if (event == "Finish"):
               try:
                   gofValue = self.metadata.get('gof')
                   gofType = self.metadata.get('gofType')
               except KeyError as e:
                   raise InvalidMeasurementError(
                       "Expected `gof` and `gofType` metadata keywords"
                       ) from e
               measurement = lsst.verify.Measurement(
                   "DcrGof",
                   gofValue * getUnits(gofType))
               measurement.combiner = getCombiner(gofType)
               measurement.notes['gofStatistic', gofType]
               self.task.job.measurements.insert(measurement)

Assuming users don't just adopt the default settings, the config file for ``DcrMatchTemplateTask`` might look something like:

.. code-block:: py

   config.listeners['DcrGoodnessOfFitMeasurer'] = EventListenerConfig()
   config.listeners['DcrGoodnessOfFitMeasurer'].events = ['Finish']

.. _arch-visitor:

Use Visitors to Make Measurements
---------------------------------

.. _arch-visitor-structure:

Architecture and Standard Components
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

In this design, ``Measurement`` objects will be made by factory objects separate from the task itself.
The factory objects are created at a high level and applied to the task hierarchy as a whole, so managing the resulting measurements can be done by a single ``Job`` object.

Measurement factories will be passed to a top-level task using a new method (``Task.accept``) after the task has completed its processing.
Each task is responsible for calling a factory's ``actOn`` method (named thus to allow for future applications other than metrics) with itself as an argument, as well as calling ``accept`` on its subtasks recursively.
The ``actOn`` method is responsible for constructing a ``Measurement`` from the information available in the completed task.
The ``Measurements`` can be stored in the factories that make them, and collected by the code that called the original ``accept`` method.

Each factory's ``actOn`` method must accept any ``Task``.
Factories for metrics that apply only to certain tasks can check the type of the argument, and do nothing if it doesn't match.
This leads to a brittle design (an unknown number of factories must be updated if an alternative to an existing task is added), but it makes adding new tasks far less difficult than a conventional visitor pattern would.

Measurements may depend on information that is internal to ``run`` or a task's other methods.
If this is the case, internal information may be stored in metadata and then extracted by the factory.

Factories can handle metrics that depend on multiple tasks (such as the :ref:`DIASource fraction<arch-visitor-examples-fdia>`) by collecting the necessary information in ``actOn``, but delaying construction of a ``Measurement`` until it is requested.
Constructing the ``Measurement`` outside of ``actOn`` is necessary because factories cannot, in general, assume that subtasks will be traversed in the order that's most convenient for them.

The correct way to merge measurements from multiple units of work depends on the metric (for example, the four use cases described :ref:`above <use-cases>` require three different approaches).
Factory classes can provide a merging function appropriate for the metric(s) they compute.
The merging can even be internal to the factory, so long as it can keep straight which measurements belong to the same task.
See :ref:`the figure below<fig-visitor-sequence>` for an example of a factory that creates measurements for both multiple tasks and multiple units of work for the same task.

.. figure:: /_static/visitor_data_flow.svg
   :name: fig-visitor-sequence
   :target: _static/visitor_data_flow.svg

   Illustration of how measurements are handled in the visitor-based architecture.
   ``anInstance`` and ``anotherInstance`` are ``ConcreteCmdLineTask`` objects run on different data.
   The subtask of ``anotherInstance`` is omitted for clarity, as are ``aFactory``'s calls to task methods.

.. _arch-visitor-workload:

Requirements for Task Creators and Maintainers
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Authors of new tasks must be aware of any metrics that apply to the new task but not to all tasks, and modify the code of applicable factories to handle the new task.
If the factories make assumptions about a task's fields, they may constrain the implementation of the task itself.

Custom task runners that call ``run`` multiple times per ``Task`` object must call ``accept`` after each run, to ensure no information is lost.
(This is not a problem for ``TaskRunner``, which creates a new ``Task`` object for each run.)

In general, maintaining instrumented tasks is easy.
The only risk is if factories constrain task implementations in some way; such details must be clearly marked as unchangeable.
If factories depend on particular metadata keys being available, the lines that write those keys must be kept in sync with the key names assumed by factories.

Authors of new metrics must implement a factory that measures them, most likely in a central verification package, and register it in a central list of metrics to be applied to tasks.
The factory implementation must consider the consequences of being passed any ``Task``, including classes that have not yet been developed.

.. _arch-visitor-procon:

Advantages and Disadvantages
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Because it is so highly centralized, the visitor-based architecture is the best at dealing with cross-task metrics -- each visitor accesses all tasks run on a particular unit of work, whether it needs to or not.

The difficulty of adding new tasks is this architecture's greatest weakness.
Neither task code nor task configurations are aware of what metrics are being applied, making it difficult for authors of new tasks to know which measurers need to know about them.
Metrics that apply to a broad category of tasks (e.g., "any task implementation that handles matching") are the most vulnerable; neither universal metrics nor implementation-specific metrics are likely to need code changes in response to new tasks.

Adding metrics always requires changes to two packages, the minimum allowed by the ``verify`` framework.
Metrics cannot be associated or disconnected from a specific task without modifying code, although the top-level registry makes it easy to globally disable a metric.

Extracting measurements from a task may require that a task write metadata it normally would not, duplicating information and forcing a task to have some knowledge of its metrics despite the lack of explicit references in the code.

.. _arch-visitor-examples:

Example Metric Implementations
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. _arch-visitor-examples-time:

Running Time
""""""""""""

The existing ``timeMethod`` decorator handles finding the running time itself, so the ``Measurement`` factory only needs to package the information.
This implementation ignores tasks that don't have the ``@timeMethod`` decorator, although this carries the risk that running time metrics defined for new tasks will silently fail.

.. code-block:: py

   class RunningTimeMeasurer(Measurer):
       def __init__(self):
           self.measurements = defaultdict(list)
           self.combiner = verify.measSum

       def actOn(task):
           try:
               start = task.metadata.get("runStartCpuTime")
               end = task.metadata.get("runEndCpuTime")
           except pexExceptions.NotFoundError:
               return
           metricName = "%s_RunTime" % type(task).__name__
           measurement = lsst.verify.Measurement(metricName,
                                                 (end - start) * u.seconds))
           self.measurements[type(task)].append(measurement)

.. _arch-visitor-examples-nastro:

Number of Sources Used for Astrometric Solution
"""""""""""""""""""""""""""""""""""""""""""""""

Astrometric tasks return the number of sources used in the fitting process, but this information is not easily available while iterating over the task hierarchy.
This implementation assumes all returned information is also stored in metadata.

This implementation also assumes that whatever central registry keeps track of ``Measurement`` factories allows constructor arguments to be specified, to minimize code duplication.

.. code-block:: py

   class SourceCounter(Measurer):
       def __init__(self, metric):
           self.measurements = defaultdict(list)
           self.combiner = verify.measSum
           self.metricName = metric

       def actOn(task):
           if isinstance(task, AstrometryTask) or isinstance(task, BetterAstrometryTask):
               try:
                   nSources = self.metadata.get('sources')
               except KeyError as e:
                   raise InvalidMeasurementError(
                       "Expected `sources` metadata keyword"
                       ) from e
               measurement = lsst.verify.Measurement(
                   self.metricName,
                   nSources * u.dimensionless_unscaled))
               self.measurements[type(task)].append(measurement)

.. _arch-visitor-examples-fdia:

Fraction of Science Sources that Are DIASources
"""""""""""""""""""""""""""""""""""""""""""""""

This metric requires combining information from ``CalibrateTask`` and ``ImageDifferenceTask``.
This implementation assumes a single, high-level task manages the entire pipeline, so that ``CalibrateTask`` and ``ImageDifferenceTask`` are indirect subtasks of it.
A similar implementation will work if ``CalibrateTask`` and ``ImageDifferenceTask`` do not share an ancestor task, but the pipeline framework must take care to pass the same factory objects to all top-level tasks.

.. code-block:: py

   class DiaFractionMeasurer(Measurer):
       def __init__(self):
           self._scienceSources = 0
           self._diaSources = 0

       def actOn(task):
           if isinstance(task, CalibrateTask):
               try:
                   self._scienceSources += task.metadata.get('sources')
               except KeyError as e:
                   raise InvalidMeasurementError(
                       "Expected `sources` metadata keyword in %s" % task
                       ) from e
           elif isinstance(task, ImageDifferenceTask):
               try:
                   self._diaSources += task.metadata.get('sources')
               except KeyError as e:
                   raise InvalidMeasurementError(
                       "Expected `sources` metadata keyword in %s" % task
                       ) from e

       # override Measurer.getMergedMeasurements()
       def getMergedMeasurements():
           # Most Measurements are not created if code not run, be consistent
           if self._scienceSources > 0:
               measurement = lsst.verify.Measurement(
                   "Fraction_DiaSource_ScienceSource",
                   (self._diaSources / self._scienceSources) * u.dimensionless_unscaled)
               return [measurement]
           else:
               return []

A cleaner implementation would be to provide an abstract subclass of ``Measurer`` that minimizes the work (and room for error) that needs to be done when developing a cross-task metric.
However, designing such a class is beyond the scope of this tech note.

.. _arch-visitor-examples-dcrgof:

DCR Goodness of Fit
"""""""""""""""""""

``DcrMatchTemplateTask`` does not yet exist, but I assume it would report goodness-of-fit in the task metadata even in the absence of a verification framework.
The factory wraps the metadata in a ``Measurement``.

.. code-block:: py

   class DcrGoodnessOfFitMeasurer(Measurer):
       def __init__(self):
           self.measurements = defaultdict(list)
           self.combiner = None

       def actOn(task):
           if isinstance(task, DcrMatchTemplateTask):
               try:
                   gofValue = self.metadata.get('gof')
                   gofType = self.metadata.get('gofType')
               except KeyError as e:
                   raise InvalidMeasurementError(
                       "Expected `gof` and `gofType` metadata keywords"
                       ) from e
               measurement = lsst.verify.Measurement(
                   "DcrGof",
                   gofValue * getUnits(gofType)))
               measurement.notes['gofStatistic', gofType]
               self.combiner = getCombiner(gofType)  # assumed same for all runs
               self.measurements[type(task)].append(measurement)

.. _comparisons:

Comparisons
===========

None of the four designs presented here satisfy all the :ref:`design goals <design-goals>`; while all four have the same basic capabilities, the more difficult aspects of the measurement problem are handled well by some architectures but not others.
The implications of each architecture for the design goals are summarized below.

Scalability to many metrics
---------------------------

- The :ref:`metadata-based architecture<arch-metadata>` requires a new decorator, per task, for each metric or group of metrics.
  In addition, the centralized code needed to merge results from multiple units of work may bloat as new kinds of metrics are introduced.
- The :ref:`direct measurement architecture<arch-direct>` requires a new decorator or function call, per task, for each metric or group of metrics.
- The :ref:`observer-based architecture<arch-observer>` requires a new config entry, per task, for each metric or group of metrics.
- The :ref:`visitor-based architecture<arch-visitor>` requires a new config entry in a central location for each metric or group of metrics.

The metadata-based architecture will scale the most poorly to large numbers of metrics, largely because of the need for long if-else chains when interpreting the metadata.
The visitor-based architecture is the best at avoiding lengthy code or configuration information.

Supporting metrics that apply to any task
-----------------------------------------

All four designs handle this case well.
The measurement code could live in ``pipe_base`` or a dependency.

Supporting metrics for groups of related tasks (such as alternate implementations)
----------------------------------------------------------------------------------

All architectures may impose API restrictions on a task that are not required by its parent task, such as producing the same metadata or sharing object attributes.

- The :ref:`metadata-based<arch-metadata>` and :ref:`direct measurement<arch-direct>` architectures require that all tasks in a group have the same ``run`` decorator.
- The :ref:`observer-based architecture<arch-observer>` requires that all tasks in a group have the same measurement factory in their configs.
- The :ref:`visitor-based architecture<arch-visitor>` requires that the metric know of all tasks in a group.

While all four architectures require that a metric be explicitly associated with each member of the group, the visitor-based architecture handles group metrics worse than the others because task authors need to dig through all metrics to find out which ones they need to support.

Supporting task-specific metrics
--------------------------------

All four designs handle this case well.
For all cases except the :ref:`visitor-based architecture<arch-visitor>`, the measurement code could live in the task package.
In the visitor-based architecture, all measurement code must be in a centralized location.

Supporting cross-task metrics
-----------------------------

- The :ref:`metadata-based architecture<arch-metadata>` requires a special channel for each task's information, and requires that ``verify_measurements`` have some custom code for assembling the final measurement.
- The :ref:`direct measurement<arch-direct>` and :ref:`observer-based<arch-observer>` architectures require either passing measurement information through metadata, or imposing restrictions on how metrics can be defined.
  A centralized handler is needed for cross-task metrics but not other metrics.
- The :ref:`visitor-based architecture<arch-visitor>` requires a nonstandard measurement factory.

The visitor-based architecture is by far the best at cross-task metrics; the direct measurement and observer-based architectures are the worst.

Associating measurements with a task class
------------------------------------------

All four designs interact with a task object, so the measurement can easily be made specific to the class if need be (the ``<class>_RunTime`` metric in the examples illustrates one way to do this).

Associating measurements with a subtask slot in a parent task
-------------------------------------------------------------

- The :ref:`metadata-based architecture<arch-metadata>` provides this information as part of the metadata key.
- The :ref:`direct measurement<arch-direct>` and :ref:`observer-based<arch-observer>` architectures can extract information about the task's relationship with its parent from the task object directly.
  In the observer-based architecture, the functionality can be hidden in a base class for factories.
- The :ref:`visitor-based architecture<arch-visitor>` architecture can extract information about the task's relationship with its parent from the task object, like an observer, or it can use config information to do so as part of a post-processing step.

The metadata-based architecture handles by-subtask metrics most naturally, but all four designs can easily provide this information.

Adding new metrics
------------------

- The :ref:`metadata-based<arch-metadata>`, :ref:`direct measurement<arch-direct>`, and :ref:`observer-based<arch-observer>` architectures require writing the appropriate measurement code, then registering it with each task of interest.
  All three designs provide workarounds to minimize the workload for widely-applicable metrics.
- The :ref:`visitor-based architecture<arch-visitor>` requires writing the appropriate measurement code, and having it test whether tasks apply to it.

Adding a universally applicable metric requires less work in the visitor-based architecture but more work in the others, while for task-specific metrics the situation is reversed.

Adding new tasks
----------------

- The :ref:`metadata-based<arch-metadata>` and :ref:`direct measurement<arch-direct>` architectures require new tasks to have the appropriate decorators for their tasks.
  In the direct measurement architecture, some metrics may require internal function calls rather than decorators, which are more difficult to spot in old tasks' code.
- The :ref:`observer-based architecture<arch-observer>` requires new tasks to have the appropriate entries in their config.
- The :ref:`visitor-based architecture<arch-visitor>` may require changes to measurement code when new tasks are added.
  The set of metrics to update cannot be determined by looking at old tasks' code.

The observer-based architecture requires slightly less work than the metadata-based or direct measurement architectures.
The visitor-based architecture is considerably worse at handling new tasks than the other three.

Allowing pipeline users to ignore metrics
-----------------------------------------

None of the four designs require user setup or force the user to handle measurements.
At worst, a ``Job`` object might be persisted unexpectedly, and persisted Jobs will become invisible once ``verify`` uses Butler persistence.

Providing measurement context
-----------------------------

- The :ref:`metadata-based architecture<arch-metadata>` can pass auxiliary information as additional keys, so long as they can be found by ``verify_measurements``.
  The :ref:`DCR goodness of fit example<arch-metadata-examples-dcrgof>` shows one way to do this.
- The :ref:`direct measurement<arch-direct>`, :ref:`observer-based<arch-observer>`, and :ref:`visitor-based<arch-visitor>` architectures all create ``Measurement`` objects on the spot, so auxiliary information can be attached using the tools provided by the ``verify`` framework.
  However, in all three cases some contextual information might be considered internal to the class, and require special handling to pass it to the code that makes the ``Measurements``.

All four designs can provide context information, although in the metadata-based architecture this comes at the cost of a more complex key naming convention.

Remaining agnostic to units of work
-----------------------------------

- The :ref:`metadata-based architecture<arch-metadata>` has a lot of difficulty reporting measurements as if all the data were processed in a single task invocation.
  Because the combining code cannot be provided by the task package, it requires cross-package coordination in a way that is bug-prone and scales poorly to large numbers of metrics.
- The :ref:`direct measurement<arch-direct>` and :ref:`observer-based<arch-observer>` architectures give ``Measurements`` the code needed to combine them.
  This code must be called either from ``CmdLineTask.parseAndRun``, or from a verification package.
- The :ref:`visitor-based architecture<arch-visitor>` give ``Measurement`` factories the code needed to combine measurements.
  This code must be called from ``CmdLineTask.parseAndRun``.

The visitor-based architecture handles the issue of units of work slightly more cleanly than the direct measurement or observer-based architectures.
The metadata-based architecture is considerably worse than the other three.

Supporting families of similar measurements
-------------------------------------------

All four architectures can handle families of metrics (e.g., running time for different task classes, or astrometric quality for different CCDs) by treating them as independent measurements.
However, in all four cases some care would need to be taken to keep the measurements straight, particularly when combining measurements of the same metric for multiple units of work.

Enabling/disabling expensive metrics
------------------------------------

- The :ref:`metadata-based<arch-metadata>` and :ref:`direct measurement<arch-direct>` architectures incorporate measurements directly into code, making it difficult to remove them completely.
  They can still check a config flag before running, however.
- The :ref:`observer-based architecture<arch-observer>` uses configs to attach measurement factories to tasks, so they can be easily added or removed.
  However, disabling calculation of a metric for all tasks requires touching many configs.
- The :ref:`visitor-based architecture<arch-visitor>` uses a central config to pass measurement factories to tasks, so they can be easily added or removed.
  However, a measurement cannot be disabled for specific tasks without modifying code.

Given that we most likely wish to disable expensive metrics globally, the visitor-based architecture provides the best support for this feature, and the observer-based architecture the worst.

Forward-compatibility with SuperTask
------------------------------------

The design described in `DMTN-055`_ makes a number of significant changes to the task framework, including
requiring that tasks be immutable (a requirement currently violated by ``Task.metadata``),
defining pipelines via a new class rather than a high-level ``CmdLineTask``,
and
introducing an ``ExecutorFramework`` for pre- and post-processing pipelines.

- The :ref:`metadata-based architecture<arch-metadata>` can be translated to SuperTask easily, once the metadata system itself is fixed to allow immutable tasks.
  The proposed ``verify_measurements`` package would be partially or wholly replaced by ``ExecutionFramework``.
- The :ref:`direct measurement architecture<arch-direct>` could be adapted by making each task's ``Job`` part of its return value rather than an attribute.
  It would make ``ExecutionFramework`` responsible for combining measurements for multiple units of work and dealing with cross-Task metrics.
- The :ref:`observer-based architecture<arch-observer>` would struggle with immutable tasks, as observers cannot alter a ``run`` method's return value the way decorators can.
  There would likely need to be a static singleton object responsible for collecting ``Measurements`` as they are made by different tasks.
  The architecture as presented would also struggle with the effective statelessness of tasks.
- The :ref:`visitor-based architecture<arch-visitor>` would require that pipelines ensure visitors are passed to each high-level task in the pipeline.
  It's not clear how this would affect the Pipeline framework's flexibility.
  The architecture would not be able to handle stateless tasks, however, as there is no other way to pass task information to a visitor.

The observer- and visitor-based architecture will adapt the worst to the SuperTask framework, while the metadata and direct-measurement architectures will have relatively little difficulty.

.. _summary:

Summary
=======

The results of the :ref:`comparisons<comparisons>` are summarized in :ref:`the table below <table-summary>`.
While no design satisfies all the design goals, the direct-measurement and visitor-based architectures come close.
The best design to pursue depends on the relative priorities of the design goals; such a recommendation is outside the scope of this tech note.

.. _table-summary:

.. table:: Each design's appropriateness with respect to the :ref:`design goals<design-goals>`.

    +---------------------------------+------------+----------+----------+---------+
    | Design Goal                     | Metadata   | Direct   | Observer | Visitor |
    +=================================+============+==========+==========+=========+
    | Scalability to many metrics     | Poor       | Fair     | Fair     | Good    |
    +---------------------------------+------------+----------+----------+---------+
    | Supporting metrics that apply to| Good       | Good     | Good     | Good    |
    | any task                        |            |          |          |         |
    +---------------------------------+------------+----------+----------+---------+
    | Supporting metrics for groups of| Fair       | Fair     | Fair     | Poor    |
    | related tasks                   |            |          |          |         |
    +---------------------------------+------------+----------+----------+---------+
    | Supporting for task-specific    | Good       | Good     | Good     | Fair    |
    | metrics                         |            |          |          |         |
    +---------------------------------+------------+----------+----------+---------+
    | Supporting cross-task metrics   | Fair       | Poor     | Poor     | Good    |
    +---------------------------------+------------+----------+----------+---------+
    | Associating measurements with a | Good       | Good     | Good     | Good    |
    | task class                      |            |          |          |         |
    +---------------------------------+------------+----------+----------+---------+
    | Associating measurements with a | Good       | Fair     | Fair     | Fair    |
    | subtask slot                    |            |          |          |         |
    +---------------------------------+------------+----------+----------+---------+
    | Adding new metrics              | Fair       | Fair     | Fair     | Fair    |
    +---------------------------------+------------+----------+----------+---------+
    | Adding new tasks                | Fair       | Fair     | Fair     | Poor    |
    +---------------------------------+------------+----------+----------+---------+
    | Allowing pipeline users to      | Good       | Fair     | Fair     | Fair    |
    | ignore metrics                  |            |          |          |         |
    +---------------------------------+------------+----------+----------+---------+
    | Providing measurement context   | Fair       | Good     | Good     | Good    |
    +---------------------------------+------------+----------+----------+---------+
    | Remaining agnostic to units of  | Poor       | Good     | Good     | Good    |
    | work                            |            |          |          |         |
    +---------------------------------+------------+----------+----------+---------+
    | Supporting families of similar  | Fair       | Fair     | Fair     | Fair    |
    | measurements                    |            |          |          |         |
    +---------------------------------+------------+----------+----------+---------+
    | Enabling/disabling expensive    | Fair       | Fair     | Poor     | Good    |
    | metrics                         |            |          |          |         |
    +---------------------------------+------------+----------+----------+---------+
    | Forward-compatibility with      | Good       | Fair     | Poor     | Poor    |
    | SuperTask                       |            |          |          |         |
    +---------------------------------+------------+----------+----------+---------+

.. .. rubric:: References

.. _DMTN-055: https://dmtn-055.lsst.io/v/DM-11523/index.html
.. _SQR-019: https://sqr-019.lsst.io/

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :encoding: latex+latin
..    :style: lsst_aa
