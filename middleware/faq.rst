##########################
Frequently asked questions
##########################

.. py:currentmodule:: lsst.daf.butler

This page contains answers to common questions about the data access and pipeline middleware, such as the `Butler` and `~lsst.pipe.base.PipelineTask` classes.
The :ref:`lsst.daf.butler` package documention includes a number of overview documentation pages (especially :ref:`daf_butler_organizing_datasets`) that provide an introduction to many of the concepts referenced here.

.. _middleware_faq_query_methods:

When should I use each of the query methods/commands?
=====================================================

The `Registry` class and :any:`butler <lsst.daf.butler-scripts>` command-line tool support five major query operations that can be used to inspect a data repository:

- `~Registry.queryCollections`
- `~Registry.queryDatasetTypes`
- `~Registry.queryDimensionRecords`
- `~Registry.queryDatasets`
- `~Registry.queryDataIds`

The :any:`butler <lsst.daf.butler-scripts>` command-line versions of these use the same names, but with dash-separated lowercase words (e.g. :any:`butler query-dimension-records <lsst.daf.butler-scripts>`).

These operations share :ref:`many optional arguments <daf_butler_queries>` that constrain what is returned, but their return types each reflect a different aspect of :ref:`how datasets are organized <daf_butler_organizing_datasets>`).

.. note::

    As a rule, these query methods return lazy iterator objects (sometimes custom classes, sometimes generators), and hence users calling them in interactive Python will often want to wrap the results in `list` or `set` in order to actually fetch results so they can be printed or iterated over multiple times::

        >>> print(butler.registry.queryDataIds("detector", instrument="HSC"))
        <DataCoordinate iterable with dimensions={instrument, detector}>

        >>> print(list(butler.registry.queryDataIds("detector", instrument="HSC")))
        {instrument: 'HSC', detector: 0},
        {instrument: 'HSC', detector: 1},
        ...



.. _middleware_faq_query_methods_collections:

queryCollections
----------------

`Registry.queryCollections` generally provides the best high-level view of the contents of a data repository, and from the command line the best way to view those high-level results is with the ``--chains=tree`` format.
For all but the smallest repos, it's best to start with some kind of guess at what you're looking for, or the results will still be overwhelming large.

.. code-block:: sh

    $ butler query-collections /repo/main HSC/runs/RC2/* --chains=tree
                     Name                      Type
    -------------------------------------- -----------
    HSC/runs/RC2/w_2021_02/DM-28282/sfm    RUN
    HSC/runs/RC2/w_2021_02/DM-28282/rest   RUN
    HSC/runs/RC2/w_2021_06/DM-28654        CHAINED
      HSC/runs/RC2/w_2021_06/DM-28654/rest RUN
      HSC/runs/RC2/w_2021_06/DM-28654/sfm  RUN
      HSC/raw/RC2/9615                     TAGGED
      HSC/raw/RC2/9697                     TAGGED
      HSC/raw/RC2/9813                     TAGGED
      HSC/calib/gen2/20180117              CALIBRATION
      HSC/calib/DM-28636                   CALIBRATION
      HSC/calib/gen2/20180117/unbounded    RUN
      HSC/calib/DM-28636/unbounded         RUN
      HSC/masks/s18a                       RUN
      skymaps                              RUN
      refcats/DM-28636                     RUN
    HSC/runs/RC2/w_2021_02/DM-28282        CHAINED
      HSC/runs/RC2/w_2021_02/DM-28282/rest RUN
      HSC/runs/RC2/w_2021_02/DM-28282/sfm  RUN
      HSC/raw/RC2/9615                     TAGGED
      HSC/raw/RC2/9697                     TAGGED
      HSC/raw/RC2/9813                     TAGGED
      HSC/calib/gen2/20180117              CALIBRATION
      HSC/calib/DM-28636                   CALIBRATION
      HSC/calib/gen2/20180117/unbounded    RUN
      HSC/calib/DM-28636/unbounded         RUN
      HSC/masks/s18a                       RUN
      skymaps                              RUN
      refcats/DM-28636                     RUN
    HSC/runs/RC2/w_2021_06/DM-28654/sfm    RUN
    HSC/runs/RC2/w_2021_06/DM-28654/rest   RUN

Note that some collections appear multiple times here - once as a top-level collection, and again later as some child of a `~CollectionType.CHAINED` collection (that's what the indentation means here).
In the future we may be able to remove some of this duplication.

queryDatasetTypes
-----------------

`Registry.queryDatasetTypes` reports the :ref:`dataset types <daf_butler_dataset_types>` that have been registered with a data repository, even if there aren't any datasets of that type actually present.
That makes it less useful for exploring a data repository generically, but it's an important tool when you know the name of the dataset type already and want to see how it's defined.

queryDimensionRecords
---------------------

`Registry.queryDimensionRecords` is the best way to inspect the metadata records associated with data ID keys (:ref:`"dimensions" <lsst.daf.butler-dimensions_overview>`), and is usually the right tool for those looking for something similar to Gen2's `~lsst.daf.persistence.Butler.queryMetadata`.
Those metadata tables include observations (the ``exposure`` and ``visit`` dimensions), instruments (``instrument``, ``physical_filter``, ``detector``), and regions on the sky (``skymap``, ``tract``, ``patch``, ``htm7``).
That isn't an exhaustive list of dimension tables (actually pseudo-tables in some cases), but you can get one in Python with::

    >>> print(butler.dimensions.getStaticDimensions())

And while `~Registry.queryDimensionRecords` shows you the schema of those tables with each record it returns, you can also get it without querying for any data with (e.g.)

.. code-block:: python

    >>> print(butler.dimensions["exposure"].RecordClass.fields)
    exposure:
      instrument: str
      id: int
      physical_filter: str
      obs_id: str
      exposure_time: float
      dark_time: float
      observation_type: str
      observation_reason: str
      day_obs: int
      seq_num: int
      group_name: str
      group_id: int
      target_name: str
      science_program: str
      tracking_ra: float
      tracking_dec: float
      sky_angle: float
      zenith_angle: float
      timespan: lsst.daf.butler.Timespan

For most dimensions and most data repositories, the number of records is quite large, so you'll almost always want a very constraining ``where`` argument to control what's returned, e.g.:

.. code-block:: sh

    $ butler query-dimension-records /repo/main detector \
        --where "instrument='HSC' AND detector.id IN (6..8)"
    instrument  id full_name name_in_raft raft purpose
    ---------- --- --------- ------------ ---- -------
           HSC   6      1_44           44    1 SCIENCE
           HSC   7      1_45           45    1 SCIENCE
           HSC   8      1_46           46    1 SCIENCE

When working with repositories of transient, cached datasets, note that dimension values may be retained in the registry for datasets that no longer exist (e.g. for provenance purposes) and may sometimes be present for datasets that do not yet exist.
As a result, you should typically constrain the results using the ``datasets`` argument and possibly the ``collections`` argument to return only values for datasets that currently exist.
Note that duplicate values may be returned (`see below <middleware_faq_duplicate_results>`_).

queryDatasets
-------------

`Registry.queryDatasets` is used to query for `DatasetRef` objects - handles that point directly to something at least approximately like a file on disk.
These correspond directly to what can be retrieved with `Butler.get`.

Because there are usually many datasets in a data repository (even in a single collection), this also isn't a great tool for general exploration; it's perhaps most useful as a way to explore things *like* the thing you're looking for (perhaps because a call to `Butler.get` unexpectedly failed), by looking with similar collections, dataset types, or data IDs.

`~Registry.queryDatasets` usually *isn't* what you want if you're looking for raw-image metadata (use `~Registry.queryDimensionRecords` instead); it's easy to confuse the dimensions that represent observations with instances of the ``raw`` dataset type, because they are always ingested into the data repository together.

queryDataIds
------------

`Registry.queryDataIds` is used to query for combinations of dimension values that *could* be used to identify datasets.

The most important thing to know about `~Registry.queryDataIds` is when *not* to use it:

- It's usually not what you want if you're looking for datasets that already exist (use `~Registry.queryDatasets` instead).
  While `~Registry.queryDataIds` lets you constrain the returned data IDs to those for which a dataset exists (via the ``datasets`` keyword argument and ``--datasets`` and ``--collections`` options), that's a subtler, higher-order thing than what most users want.

- It's usually not what you want if you're looking for metadata associated with those data ID values (use `~Registry.queryDimensionRecords`).
  While `~Registry.queryDataIds` can do that, too (via the `~registry.queries.DataCoordinateQueryResults.expanded` method on its result iterator), it's overkill if you're looking for metadata that corresponds to a single dimension rather than all of them.

`~Registry.queryDataIds` is most useful when you want to query for future datasets that *could* exist, such as when :ref:`debugging empty QuantumGraphs <middleware_faq_empty_quantum_graphs>`.

.. _middleware_faq_cli_docs:

Where can I find documentation for command-line butler queries?
===============================================================

The ``butler`` command line tool uses a plugin system to allow packages downstream of ``daf_butler`` to define their own ``butler`` subcommands.
Unfortunately, this means there's no single documentation page that lists all subcommands; each package has its own page documenting the subcommands it provides.
The :ref:`daf_butler <lsst.daf.butler-scripts>` and :ref:`obs_base <lsst.obs.base-cli>` pages contain most subcommands, but the best way to find them all is to use ``--help`` on the command-line.

The :any:`pipetask <lsst.ctrl.mpexec-script>` tool is implemented entirely within ``ctrl_mpexec``, and its documentation can be found on :ref:`the command-line interface page for that package <lsst.ctrl.pipetask-script>` (and of course via ``--help``).

.. _middleware_faq_duplicate_results:

Why do queries return duplicate results?
========================================

The `Registry.queryDataIds`, `~Registry.queryDatasets`, and `~Registry.queryDimensionRecords` methods can sometimes return true duplicate values, simply because the SQL queries used to implement them do.
You can always remove those duplicates by wrapping the calls in ``set()``; the `DataCoordinate`, `DatasetRef`, and `DimensionRecord` objects in the returned iterables are all hashable.
This is a conscious design choice; these methods return lazy iterables in order to handle large results efficiently, and that rules out removing duplicates inside the methods themselves.
We similarly don't want to *always* remove duplicates in SQL via ``SELECT DISTINCT``, because that can be much less efficient than deduplication in Python, but in the future we may have a way to turn this on explicitly (and may even make it the default).
We do already remove these duplicates automatically in the :any:`butler <lsst.daf.butler-scripts>` command-line interface.

It is also possible for `~Registry.queryDatasets` (and the :any:`butler query-datasets <lsst.daf.butler-scripts>` command) to return datasets that have the same dataset type and data ID from different collections,
This can happen even if the users passes only collection to search, if that collection is a `~CollectionType.CHAINED` collection (because this evaluates to searching one or more child collections).
These results are not true duplicates, and will not be removed by wrapping the results in ``set()``.
They are best removed by passing ``findFirst=True`` (or ``--find-first``), which will return - for each data ID and dataset type - the dataset from the first collection with a match.
For example, from the command-line, this command returns one ``calexp`` from each of the given collections:

..
    Can't use prompt:: directive here instead because it can't handle program output.

.. code-block:: sh

    $ butler query-datasets /repo/main calexp \
        --collections HSC/runs/RC2/w_2021_06/DM-28654 \
        --collections HSC/runs/RC2/w_2021_02/DM-28282 \
        --where "instrument='HSC' AND visit=1228 AND detector=40"

     type                  run                    id   band instrument detector physical_filter visit_system visit
    ------ ----------------------------------- ------- ---- ---------- -------- --------------- ------------ -----
    calexp HSC/runs/RC2/w_2021_02/DM-28282/sfm 5928697    i        HSC       40           HSC-I            0  1228
    calexp HSC/runs/RC2/w_2021_06/DM-28654/sfm 5329565    i        HSC       40           HSC-I            0  1228

(with no guaranteed order!) while adding ``--find-first`` yields only the ``calexp`` found in the first collection:

.. code-block:: sh

    $ butler query-datasets /repo/main calexp --find-first \
        --collections HSC/runs/RC2/w_2021_06/DM-28654 \
        --collections HSC/runs/RC2/w_2021_02/DM-28282 \
        --where "instrument='HSC' AND visit=1228 AND detector=40"

    type                  run                    id   band instrument detector physical_filter visit_system visit
    ------ ----------------------------------- ------- ---- ---------- -------- --------------- ------------ -----
    calexp HSC/runs/RC2/w_2021_06/DM-28654/sfm 5329565    i        HSC       40           HSC-I            0  1228

Passing ``findFirst=True`` or ``--find-first`` requires the list of collections to be clearly ordered, however, ruling out wildcards like ``...`` ("all collections"), globs, and regular expressions.
Single-dataset search methods like `Butler.get` and `Butler.find_dataset` always use the find-first logic (and hence always require ordered collections).

.. _middleware_faq_data_id_missing_keys:

Why are some keys (usually filters) sometimes missing from data IDs?
====================================================================

While most butler methods accept regular dictionaries as data IDs, internally we standardize them into instances of the `DataCoordinate` class, and that's also what will be returned by `Butler` and `Registry` methods.
Printing a `DataCoordinate` can sometimes yield results with a confusing ``...`` in it::

    >>> dataId = butler.registry.expandDataId(instrument="HSC", exposure=903334)
    >>> print(dataId)
    {instrument: 'HSC', exposure: 903334, ...}

And similarly asking for its ``keys`` doesn't show everything you'd expect (same for ``values`` or ``items``); in particular, there are no ``physical_filter`` or ``band`` keys here, either::

    >>> print(dataId.keys())
    {instrument, exposure}

The quick solution to these problems is to use `DataCoordinate.full`, which is another more straightforward `~collections.abc.Mapping` that contains all of those keys:

    >>> print(dataId.full)
    {band: 'r', instrument: 'HSC', physical_filter: 'HSC-R', exposure: 903334}

You can also still use expressions like ``dataId["band"]``, even though those keys *seem* to be missing:

    >>> print(dataId["band"])
    r

The catch is these solutions only work if `DataCoordinate.hasFull` returns `True`; when it doesn't, accessing `DataCoordinate.full` will raise `AttributeError`, essentially saying that the `DataCoordinate` doesn't know what the filter values are, even though it knows other values (i.e. the ``exposure`` ID) that could be used to fetch them.
The terminology we use for this is that ``{instrument, exposure}`` are the *required* dimensions for this data ID and ``{physical_filter, band}`` are *implied* dimensions::

    >>> dataId.graph.required
    {instrument, exposure}
    >>> dataId.graph.implied
    {band, physical_filter}

The good news is that any `DataCoordinate` returned by the `Registry` query methods will always have `~DataCoordinate.hasFull` return `True`, and you can use `Registry.expandDataId` to transform any other `DataCoordinate` or `dict` data ID into one that contains everything the database knows about those values.

The obvious follow-up question is why `DataCoordinate.keys` and stringification don't just report all of they key-value pairs the object actually knows, instead of hiding them.
The answer is that `DataCoordinate` is trying to satisfy a conflicting set of demands on it:

- We want it to be a `collections.abc.Mapping`, so it behaves much like the `dict` objects often used informally for data IDs.
- We want a `DataCoordinate` that *only* knows the value for required dimensions to compare as equal to any data ID with the same values for those dimensions, regardless of whether those other data IDs also have values for implied dimensions.
- `collections.abc.Mapping` defines equality to be equivalent to equality over ``items()``, so if one mapping includes more keys than the other, they can't be equal.

Our solution was to make it so `DataCoordinate` is always a `~collections.abc.Mapping` over just its required keys, with ``full`` available sometimes as a `~collections.abc.Mapping` over all of them.
And because the `~collections.abc.Mapping` interface doesn't prohibit us from allowing ``__getitem__`` to succeed even when the given value isn't in ``keys``, we support that for implied dimensions as well.
It's possible it would have been better to just not make it a `~collections.abc.Mapping` at all (i.e. remove ``keys``, ``values``, and ``items`` in favor of other ways to access those things).
`DataCoordinate` :ref:`has already been through a number of revisions <lsst.daf.butler-dev_data_coordinate>`, though, and it's not clear it's worth yet another try.

.. _middleware_faq_calibration_query_errors:

How do I avoid errors involving queries for calibration datasets?
=================================================================

`Registry.queryDatasets` currently has a major limitation in that it can't query for datasets within a `~CollectionType.CALIBRATION` collection; the error message looks like this::

    NotImplementedError: Query for dataset type 'flat' in CALIBRATION-type collection 'HSC/calib' is not yet supported.

We do expect to fix this limitation in the future, but it may take a while.
In the meantime, there are a few ways to work around this problem.

First, if you don't actually want to search for calibrations at all, but this exception is still getting in your way, you can make your query more specific.
If you use a dataset type list or pattern (a shell-style glob on the command line, or `re.compile` in the Python interface) that doesn't match any calibration datasets, this error should not occur.

Similarly, if you can use a list of collections or a collection pattern that doesn't include any `~CollectionType.CALIBRATION` collections, that will avoid the problem as well - but this is harder, because `~CollectionType.CHAINED` collections that include `~CollectionType.CALIBRATION` collections are quite common.
For example, both processing-output collections with names like "HSC/runs/w_2025_06/DM-50000" and per-instrument default collections like "HSC/defaults" include a `~CollectionType.CALIBRATION` child collection.
You can recursively expand a collection list and filter out any child `~CollectionType.CALIBRATION` collections from it with this snippet::

    expanded = list(
        butler.registry.queryCollections(
            original,
            flattenChains=True,
            collectionTypes=(CollectionType.all - {CollectionType.CALIBRATION}),
        )
    )

where ``original`` is the original, unexpanded list of collections to search.

The equivalent command-line invocation is:

.. code-block:: sh

    $ butler query-collections /repo/main --chains=flatten \
            --collection-type RUN \
            --collection-type CHAINED \
            --collection-type TAGGED \
            HSC/defaults
        Name               Type
    --------------------------------- ----
    HSC/raw/all                       RUN
    HSC/calib/gen2/20180117/unbounded RUN
    HSC/calib/DM-28636/unbounded      RUN
    HSC/masks/s18a                    RUN
    refcats/DM-28636                  RUN
    skymaps                           RUN

Another possible workaround is to make the query much more general - passing ``collections=...`` to search *all* collections in the repository will avoid this limitation even for calibration datasets, because it will take advantage of the fact that all datasets are in exactly one `~CollectionType.RUN` collection (even if they can also be in one or more other kinds of collection) by searching only all of the `~CollectionType.RUN` collections.

That same feature of `~CollectionType.RUN` collections can also be used with `Registry.queryCollections` (and our naming conventions) to find calibration datasets that *might* belong to particular `~CollectionType.CALIBRATION` collections.
For example, if "HSC/calib" is a `~CollectionType.CALIBRATION` collection (or a pointer to one), the datasets in it will usually also be present in `~CollectionType.RUN` collections that start with "HSC/calib/", so logic like this might be useful::

    run_collections = list(
        butler.registry.queryCollections(
            re.compile("HSC/calib/.+"),
            collectionTypes={CollectionTypes.RUN},
        )
    )

Or, from the command-line,

.. code-block:: sh

    $ butler query-collections /repo/main --collection-type RUN \
            HSC/calib/gen2/20200115/*
                    Name                   Type
    ---------------------------------------- ----
    HSC/calib/gen2/20200115/20170821T000000Z RUN
    HSC/calib/gen2/20200115/20160518T000000Z RUN
    HSC/calib/gen2/20200115/20170625T000000Z RUN
    HSC/calib/gen2/20200115/20150417T000000Z RUN
    HSC/calib/gen2/20200115/20181207T000000Z RUN
    HSC/calib/gen2/20200115/20190407T000000Z RUN
    HSC/calib/gen2/20200115/20150407T000000Z RUN
    HSC/calib/gen2/20200115/20160114T000000Z RUN
    HSC/calib/gen2/20200115/20170326T000000Z RUN
    ...

The problem with this approach is that it may return many datasets that aren't in "HSC/calib", including datasets that were not certified, and (like all of the previous workarounds) it doesn't tell you anything about the validity ranges of the datasets that it returns.

If you just want to load the calibration dataset appropriate for a particular ``raw`` (and you have the data ID for that ``raw`` in hand), the right solution is to use `Butler.get` with that raw data ID, which takes care of everything for you::

    flat = butler.get(
        "flat",
        instrument="HSC", exposure=903334, detector=0,
        collections="HSC/calib"
    )

The lower-level `Butler.find_dataset` method can also perform this search without actually reading the dataset, but you'll need to be explicit about how to do the temporal lookup::

    raw_data_id = butler.registry.expandDataId(
        instrument="HSC",
        exposure=903334,
        detector=0,
    )
    ref = butler.find_dataset(
        "flat",
        raw_data_id,
        timespan=raw_data_id.timespan,
    )

It's worth noting that `~Butler.find_dataset` doesn't need or use the ``exposure`` key in the ``raw_data_id`` argument that is passed to it - a master flat isn't associated with an exposure - but it's happy to ignore it, and we *do* need it (or something else temporal) in order to get a data ID with a timespan for the last argument.

Finally, if you need to query for calibration datasets *and* their validity ranges, and don't have a point in time you're starting from, the only option is `Registry.queryDatasetAssociations`.
That's a bit less user-friendly - it only accepts one dataset type at a time, and doesn't let you restrict the data IDs at all - but it *can* query `~CollectionType.CALIBRATION` collections and it returns the associated validity ranges as well.
It actually only exists as a workaround for the fact that `~Registry.queryDatasets` can't do those things, and it will probably be removed sometime after those limitations are lifted.

.. _middleware_faq_empty_quantum_graphs:

How do I fix an empty QuantumGraph?
===================================

.. py:currentmodule:: lsst.pipe.base

The :any:`pipetask <lsst.ctrl.mpexec-script>` tool attempts to predict all of the processing a pipeline will perform in advance, representing the results as a `QuantumGraph` object that can be saved or directly executed.
When that graph is empty, it means it thinks there's no work to be done, and unfortunately this is both a common and hard-to-diagnose problem.

The `QuantumGraph` generation algorithm begins with a large SQL query (a complicated invocation of `Registry.queryDataIds`, actually), where the result rows are essentially data IDs and the result columns are all of the dimensions referenced by any task or dataset type in the pipeline.
Queries for all `"regular input" <connectionTypes.Input>` datasets (i.e. not `PrerequisiteInputs <connectionTypes.PrerequisiteInput>`") are included as subqueries, spatial and temporal joins are automatically included, and the user-provided query expression is translated into an equivalent SQL ``WHERE`` clause.
That means there are many ways to get no result rows - and hence an empty graph.

Sometimes we can tell what will go wrong even before the query is executed - the butler maintains a summary of which dataset types are present each each collection, so if the input collections don't have any datasets of a needed type at all, a warning log message will be generated stating the problem.
This will also catch most cases where a pipeline is misconfigured such that what should be an intermediate dataset isn't actually being produced in the pipeline, because it will appear instead as an overall input that (usually) won't be present in those input collections.

We also perform some follow-up queries after generating an empty `QuantumGraph`, to see if any needed dimensions are lacking records entirely (the most common example of this case is forgetting to define visits after ingesting raws in a new data repository).

If you get an empty `QuantumGraph` without any clear explanations in the  warning logs, it means something more complicated went wrong in that initial query, such as the input datasets, available dimensions, and boolean expression being mutually inconsistent (e.g. not having any bands in common, or tracts and visits not overlapping spatially).
In this case, the arguments to `~Registry.queryDataIds` will be logged again as warnings), and the next step in debugging is to try that call manually with slight adjustments.

To guide this process, it can be very helpful to first use :any:`pipetask <lsst.ctrl.mpexec-script>` to create a diagram of the pipeline graph - a simpler directed acyclic graph that relates tasks to dataset types, without any data IDs.
The ``--pipeline-dot`` argument writes this graph in the `GraphViz dot language`_, and you can use the ubiquitous ``dot`` command-line tool to transform that into a PNG, SVG, or other graphical format file:

.. code:: sh

    $ pipetask build ... --pipeline-dot pipeline.dot
    $ dot pipeline.dot -Tsvg > pipeline.svg

That ``...`` should be replaced by most of the arguments you'd pass to :any:`pipetask <lsst.ctrl.mpexec-script>` that describe *what* to run (which tasks, pipelines, configuration, etc.), but not the ones that describe how, or what to use as inputs (no collection options).
See ``pipetask build --help`` for details.

This graph will often reveal some unexpected input dataset types, tasks, or relationships between the two that make it obvious what's wrong.

Another useful approach is to try to simplify the pipeline, ideally removing all but the first task; if that works, you can generally rule it out as the cause of the problem, add the next task in, and repeat.

Because the big initial query only involves regular inputs, it can also be helpful to change regular `~connectionTypes.Input` connections into `~connectionTypes.PrerequisiteInput` connections - when a prerequisite input is missing, :any:`pipetask <lsst.ctrl.mpexec-script>` should provide more useful diagnostics.
This is only possible when the dataset type is already in your input collections, rather than something to be produced by another task within the same pipeline.
But if you work through your pipeline task-by-task, and run each single-task pipeline as well as produce a `QuantumGraph` for it, this should be true each step of the way as well.

.. _GraphViz dot language: https://graphviz.org/


.. _middleware_faq_long_query:

What do I do if a query method/command or pipetask graph generation is slow?
============================================================================

Adding the ``--log-level sqlalchemy.engine=DEBUG`` option to the :any:`butler <lsst.daf.butler-scripts>` or :any:`pipetask <lsst.ctrl.mpexec-script>` command will allow the SQL queries issued by the command to be inspected.
Similarly, for a slow query method, adding ``logging.getLogger("sqlalchemy.engine").setLevel(logging.DEBUG)`` can help.
The resulting query logs can be useful for developers and database administrators to determine what, if anything, is going wrong.

.. _middleware_faq_clean_up_runs:

How do I clean up processing runs I don't need anymore?
=======================================================

.. py:currentmodule:: lsst.daf.butler

Because a data repository stores information on both a filesystem or object store and a SQL database, deleting datasets completely requires using butler commands, even if you know where the associated files are stored on disk.

For processing runs that follow our usual conventions (following them is automatic if you use ``--output`` and don't override ``--output-run`` when running :any:`pipetask <lsst.ctrl.mpexec-script>`), two different collections are created:

- a `~CollectionType.RUN` collection that directly holds your outputs
- a `~CollectionType.CHAINED` collection that points to that RUN collection as well as all of your input collections.

If you perform multiple processing runs with the same ``--output``, you'll get multiple `~CollectionType.RUN` collections in the same `~CollectionType.CHAINED` collection.
The `~CollectionType.CHAINED` collection will have the name you passed to ``--output``, and the RUN collections will start with that and end with a timestamp.
You can see this structure for your own collections with a command like this one:

.. code:: sh

    $ butler query-collections /repo/main --chains=tree u/jbosch/*
    u/jbosch/DM-30649                                    CHAINED
      u/jbosch/DM-30649/20210614T191615Z                 RUN
      HSC/raw/RC2/9813                                   TAGGED
      HSC/calib/gen2/20180117                            CALIBRATION
      HSC/calib/DM-28636                                 CALIBRATION
      HSC/calib/gen2/20180117/unbounded                  RUN
      HSC/calib/DM-28636/unbounded                       RUN
      HSC/masks/s18a                                     RUN
      HSC/fgcmcal/lut/RC2/DM-28636                       RUN
      refcats/DM-28636                                   RUN
      skymaps                                            RUN
    u/jbosch/DM-30649/20210614T191615Z                   RUN

The RUN collections that directly hold the datasets are what we want to remove in order to free up space, but we have to start by deleting the `~CollectionType.CHAINED` collections that hold them first:

.. code:: sh

    $ butler remove-collections /repo/main u/jbosch/DM-30649

You can add the ``--no-confirm`` option to skip the confirmation prompt if you like.

If you're only deleting one collection at a time, it doesn't tell you anything new.

Not deleting the CHAINED collection
-----------------------------------

If you don't want to remove the `~CollectionType.CHAINED` collection - you just want to remove the `~CollectionType.RUN` collection from it - you can instead do

    $ butler collection-chain /repo/main --remove u/jbosch/DM-30649 u/jbosch/DM-20210614T191615Z

Or, if you know the `~CollectionType.RUN` is the first one in the chain,

    $ butler collection-chain /repo/main --pop u/jbosch/DM-30649

In any case, once the `~CollectionType.CHAINED` collection is out of the way, we can delete the `~CollectionType.RUN` collections that start with the same prefix using a glob pattern:

.. code:: sh

    $ butler remove-runs /repo/main u/jbosch/DM-30649/*
    The following RUN collections will be removed:
    u/jbosch/DM-30649/20210614T191615Z
    The following datasets will be removed:
    calexp(18222), calexpBackground(18222), calexp_camera(168), calibrate_config(1), calibrate_metadata(18222), characterizeImage_config(1), characterizeImage_metadata(18231), consolidateSourceTable_config(1), consolidateVisitSummary_config(1), consolidateVisitSummary_metadata(168), fgcmBuildStarsTable_config(1), fgcmFitCycle_config(1), fgcmOutputProducts_config(1), icExp(18231), icExpBackground(18231), icSrc(18231), icSrc_schema(1), isr_config(1), isr_metadata(18232), postISRCCD(18232), skyCorr(17304), skyCorr_config(1), skyCorr_metadata(168), source(18222), src(18222), srcMatch(18222), srcMatchFull(18222), src_schema(1), transformSourceTable_config(1), visitSummary(168), writeSourceTable_config(1), writeSourceTable_metadata(18222)
    Continue? [y/N]: y
    Removed collections

Here we've left the default confirmation behavior on because we used a glob, just to be safe.
You can write one or more full RUN collection names explicitly (separated by commas), too, and that's what you'll need to do if you didn't follow the naming convention well enough for a glob to work.

Removing `~CollectionType.RUN` collections always removes the files within them, but it does not remove the directory structure, because in the presence of arbitrary path templates (including any that may have been used in the past) and possible concurrent writes, it's difficult for the butler to recognize efficiently when a directory will end up empty.
You're welcome to delete empty directories on your own after using ``remove-runs``; they're typically in subdirectories of the main repository directory named after the collection (it's possible to configure the butler such that this isn't the case, but rare).
It's also completely fine to just leave them.

.. note::

    If you delete files from the filesystem before using butler commands to remove entries from the database, the commands for cleaning up the database are actually exactly the same.
    The butler won't know that the files are gone until you try to use or delete them, but when you try to delete them, it will just log this at debug level.

Deleting only some datasets
---------------------------

If you don't want to delete the full RUN collection, just some datasets within it, you can generally use the ``prune-datasets`` subcommand:

.. code:: sh

    $ butler prune-datasets /repo/main --purge u/jbosch/DM-29776/singleFrame/20210426T161854Z --datasets postISRCCD u/jbosch/DM-29776/singleFrame/20210426T161854Z
    The following datasets will be removed:

    type                         run                                        id                  band instrument detector physical_filter exposure
    ---------- ---------------------------------------------- ------------------------------------ ---- ---------- -------- --------------- --------
    postISRCCD u/jbosch/DM-29776/singleFrame/20210426T161854Z c45a177f-24e8-4dc9-9268-5895decb7989    y        HSC        0           HSC-Y      318
    postISRCCD u/jbosch/DM-29776/singleFrame/20210426T161854Z 461d0293-3c80-45ea-9f06-21a90525c185    y        HSC        1           HSC-Y      318
    postISRCCD u/jbosch/DM-29776/singleFrame/20210426T161854Z 1572dd02-c959-4d23-ba03-91cf235e1291    y        HSC        2           HSC-Y      318
    postISRCCD u/jbosch/DM-29776/singleFrame/20210426T161854Z b38afec9-1970-478d-80d8-4f61c5a992d0    y        HSC        3           HSC-Y      318
    postISRCCD u/jbosch/DM-29776/singleFrame/20210426T161854Z 769bb9ce-9267-4e57-812f-82fee3fd0afa    y        HSC        4           HSC-Y      318
    (...)
    Continue? [y/N]: y
    The datasets were removed.

Note that here you have to know the exact `~CollectionType.RUN` collection that holds the datasets, and specify it twice (the argument to ``--purge`` is the collection to delete from, while the positional argument is the collection to query within - the latter could be some other kind of collection, but it's rare for that to be useful).

The Python `Butler.pruneDatasets` method can be used for even greater control of what you want to delete, as it accepts an arbitrary `DatasetRef` iterable indicating what to delete.
