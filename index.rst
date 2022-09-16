:tocdepth: 1

.. sectnum::

Abstract
========

Currently we generate ObsCore tables from Registry data with a special application which uses Registry API to discover the data.
This application needs to be run regularly to reflect Registry updates, which represent additional step in making ObsCore data available to users.
This note discusses alternative ways to extract ObsCore data from Registry that do not require running export application.


ObsCore export script
=====================

Current procedure for export of Registry data to ObsCore format is implemented in a `dax_obscore`_ package as a stand-alone application that reads Registry and generates CSV or Parquet files.
At the very high level the export application performs following operations:

- It is configured with a list of collections and dataset types to read from Registry.
- For each dataset type it calls ``Registry.queryDatasets(dataset_type_name, collections, ...)``.
- Returned datasets are expanded to include all dimension records.
- From the expanded ``DataId`` the script builds a corresponding ObsCore record.
  There are some non-trivial transformations performed during this stage:

  - Some of the ObsCore columns are filled from the configuration, either global, per-dataset, or per-filter band.
  - As the Registry does not define per-exposure regions, the script extracts exposure region from a matching visit (or visit+detector).
  - Registry stores ``sphgeom`` regions in an opaque byte-encoded format, for ObsCore the script transforms regions into an ADQL ``POLYGON`` string representation.

Additional post-processing is performed when the generated data is ingested into QServ/MySQL.
To accelerate point-in-region searches in the ObsCore table, a new spatially-indexed column is added to the table which defines a bounding box for a region.

Export performance
------------------

The ``Registry.queryDatasets`` method performs a non-trivial queries which has performance implications.
As an example below are some numbers obtained with a recent export of ``DP0.2`` Registry at ``data-int.lsst.cloud``.
Configuration included several dataset types and a chained collection "2.2i/runs/DP0.2", the collection resolves to 286 run-type collections.

============================ =========== ============
Dataset type                 Export time Record count
============================ =========== ============
raw                            15m30s    2,870,807
calexp                         16m30s    2,805,017
deepCoadd_calexp               14s          46,158
goodSeeingCoadd                11s          46,158
goodSeeingDiff_differenceExp   19m05s    2,707,834
**Total**                      51m30s
============================ =========== ============

Approximately 90% of the total time is spent on client side CPU, this is the price to pay for iterating over millions of result rows and performing non-trivial operations on each row.
These results were obtained with a ``findFirst`` parameter of ``Registry.queryDatasets`` set to ``False``.
A naturally correct value for that parameter should be ``True``, which would avoid possible duplicates in the output.
Unfortunately, setting ``findFirst`` to ``True`` increases export time dramatically, export takes approximately 36 hours to finish with that option.


SQL queries
-----------

Each call to a ``Registry.queryDatasets`` method executes a small number of SQL queries - one query  to find all dataset references and followup queries to expand references with dimension records.
If the input collection has CHAINED type, then the queries need to filter all included RUN-type collections which can grow to a very long list.
Here is an example of a query for finding references ``calexp`` dataset type with ``findFirst=False`` (collection list is truncated, it has 58 run collections in total)::

  SELECT
      physical_filter.band AS band,
      physical_filter.instrument AS instrument,
      calexp.detector AS detector,
      physical_filter.name AS physical_filter,
      visit.visit_system AS visit_system,
      visit.id AS visit,
      calexp.id AS dataset_id,
      calexp.run_name AS run_name
  FROM
      (
          SELECT
              dataset_tags_00000007.instrument AS instrument,
              dataset_tags_00000007.detector AS detector,
              dataset_tags_00000007.visit AS visit,
              dataset_tags_00000007.dataset_id AS id,
              dataset.run_name AS run_name,
              dataset.ingest_date AS ingest_date
          FROM
              dataset_tags_00000007
              JOIN dataset ON dataset_tags_00000007.dataset_id = dataset.id
          WHERE
              dataset_tags_00000007.dataset_type_id = 41
              AND dataset_tags_00000007.collection_name IN (
                  '2.2i/runs/DP0.2/v23_0_0_rc5/PREOPS-905/20220112T143133Z',
                  '2.2i/runs/DP0.2/v23_0_0_rc5/PREOPS-905/20220112T114604Z',
                  -- 55 other collection names
                  '2.2i/runs/DP0.2/v23_0_0_rc5/PREOPS-905/20211218T002844Z'
              )
              AND dataset.dataset_type_id = 41
      ) AS calexp
      JOIN visit ON calexp.instrument = visit.instrument
      AND calexp.visit = visit.id
      JOIN physical_filter ON calexp.instrument = physical_filter.instrument
      AND visit.instrument = physical_filter.instrument
      AND visit.physical_filter = physical_filter.name;

When ``findFirst`` is set to ``True`` the queries are more complicated as they need to return a unique references based on collection ordering.
Here is an example of a query with ``findFirst=False`` with a large fraction of sub-queries truncated:

.. _find-first SQL listing:

::

  WITH calexp_search AS (
      SELECT
          dataset_tags_00000007.instrument AS instrument,
          dataset_tags_00000007.detector AS detector,
          dataset_tags_00000007.visit AS visit,
          dataset_tags_00000007.dataset_id AS id,
          '2.2i/runs/DP0.2/v23_0_0_rc5/PREOPS-905/20220112T143133Z' AS run_name,
          dataset.ingest_date AS ingest_date,
          0 AS rank
      FROM
          dataset_tags_00000007
          JOIN dataset ON dataset_tags_00000007.dataset_id = dataset.id
      WHERE
          dataset_tags_00000007.dataset_type_id = 41
          AND dataset_tags_00000007.collection_name = '2.2i/runs/DP0.2/v23_0_0_rc5/PREOPS-905/20220112T143133Z'
          AND dataset.dataset_type_id = 41
      UNION
      SELECT
          dataset_tags_00000007.instrument AS instrument,
          dataset_tags_00000007.detector AS detector,
          dataset_tags_00000007.visit AS visit,
          dataset_tags_00000007.dataset_id AS id,
          '2.2i/runs/DP0.2/v23_0_0_rc5/PREOPS-905/20220112T114604Z' AS run_name,
          dataset.ingest_date AS ingest_date,
          1 AS rank
      FROM
          dataset_tags_00000007
          JOIN dataset ON dataset_tags_00000007.dataset_id = dataset.id
      WHERE
          dataset_tags_00000007.dataset_type_id = 41
          AND dataset_tags_00000007.collection_name = '2.2i/runs/DP0.2/v23_0_0_rc5/PREOPS-905/20220112T114604Z'
          AND dataset.dataset_type_id = 41
      UNION
      -- plus 56 other queries
  )
  SELECT
      physical_filter.band AS band,
      physical_filter.instrument AS instrument,
      calexp.detector AS detector,
      physical_filter.name AS physical_filter,
      visit.visit_system AS visit_system,
      visit.id AS visit,
      calexp.id AS dataset_id,
      calexp.run_name AS run_name
  FROM
      (
          SELECT
              calexp_window.id AS id,
              calexp_window.run_name AS run_name,
              calexp_window.ingest_date AS ingest_date,
              calexp_window.instrument AS instrument,
              calexp_window.detector AS detector,
              calexp_window.visit AS visit,
              calexp_window.rownum AS rownum
          FROM
              (
                  SELECT
                      calexp_search.id AS id,
                      calexp_search.run_name AS run_name,
                      calexp_search.ingest_date AS ingest_date,
                      calexp_search.instrument AS instrument,
                      calexp_search.detector AS detector,
                      calexp_search.visit AS visit,
                      row_number() OVER (
                          PARTITION BY calexp_search.instrument,
                          calexp_search.detector,
                          calexp_search.visit
                          ORDER BY
                              calexp_search.rank
                      ) AS rownum
                  FROM
                      calexp_search
              ) AS calexp_window
          WHERE
              calexp_window.rownum = 1
      ) AS calexp
      JOIN visit ON calexp.instrument = visit.instrument
      AND calexp.visit = visit.id
      JOIN physical_filter ON calexp.instrument = physical_filter.instrument
      AND visit.instrument = physical_filter.instrument
      AND visit.physical_filter = physical_filter.name;


Ideas for alternative implementations
======================================

There may be few options for supporting ObsCore functionality that do not rely on export via ``Registry.queryDatasets`` method.
These options and their related issues are discussed in sections below.


Online view of Registry tables
------------------------------

A natural idea for avoiding regular manual exports of the data is to create a SQL view that will execute the same set of queries as the export script.
It should be possible to join the query from the `find-first SQL listing`_ with the additional queries to extract dimension records and represent the result as ObsCore schema.
There are several issues with this approach though:

- The performance of the queries.
  To be usable for interactive work the queries in the view need to be executed very quick, which is not the case for a correct ``find-first`` approach.
- Chained collection management.
  The list of the run collections in a chained collection will be growing and view needs to be aware of that.
  Either the view needs to be redefined each time a new run collection is added, or view has to be made smarter to use build list of run collections at execution time.
  It may be possible to implement latter case at the expense of even more complicated queries.
- Spatial transformations.
  To represent region data in ObsCore ADQL format the view needs to decode binary ``sphgeom`` format and translate it into a different representation.
  This may be very non-trivial to implement in pure SQL.
- Indexing.
  Queries on ObsCore data will likely use restrictions on some columns.
  Without proper indexing these queries may be inefficient.
  Queries on view could in some cases use indices on underlying tables, but with the very complex queries there is no guarantee the query optimizer can do it in our case.


Materialized views
------------------

PostgreSQL supports materialized views which is a real table containing snapshot of the view query.
The materialized vew is refreshed regularly bu running a special 'REFRESH MATERIALIZED VIEW' command which re-populates the snapshot with the current data.
Because a materialized view is backed by an actual table, the indices can be defined on that table and that should help with the query performance on the view.

The issues with the performance of the view queries still has an impact with the materialized view.
The REFRESH command needs to run the whole set ot queries which presently takes very long time, so the updates cannot happen frequently.
Out-of-box PostgreSQL does not support incremental updates, though there is a work in progress to add `incremental vew update`_ support in the future.
Even if incremental update support is added, it is very doubtful that it can handle queries of the complexity that we define for our data.

Just as with the online views, collection management is also an issue with the materialized views.
Similarly the decoding of region data is an issue for any pure-SQL-based solution.


ObsCore table with triggers
---------------------------

One way to avoid running expensive queries on Registry is to insert records into an actual ObsCore table at the same time as populating regular Registry tables.
At the database level this can be implemented with INSERT/DELETE triggers on relevant Registry tables.
In PostgreSQL triggers can be implemented in a number of languages, including `PL/Python`_.

Triggers have their own set of issues that make their use non-trivial:

- PL/Python is implemented as a separate extension that needs to to be installed in each database.
- Server-side environment could be more restricted, for example, to import ``sphgeom`` package it may need to be installed separately on server host.
- Handling of collection ordering may require non-trivial logic and queries.
  Although with some assumptions the ordering check may not be necessary during INSERT.
- Performance needs special consideration, calling additional potentially complex code for each inserted row can make things slower.
- Error and exception handling in server-side functions can be more challenging.


ObsCore table with client-side updates
--------------------------------------

A different INSERT-time approach could be implemented with client-side hooks.
The Registry can be extended to update an additional ObsCore table while doing regular updates to its own schema. As usual, there are benefits and drawbacks in this approach:

- ObsCore can be thought of as a part of Registry schema which can be covered by the existing modular "managers" design, extending that design to include ObsCore could be a natural extension.
- ObsCore schema could use the same schema migration tools as other Registry managers.
- ObsCore schema may be less stable, at least initially, compared to the rest of the Registry schema, which could require more frequent schema migrations.
- There could be multiple ObsCore schemas in the same Registry, e.g. it may make sense to make separate ObsCore tables for each instrument.

There could also be a combined approach which stores stable parts of ObsCore schema in the table, and implements other less stable parts of the schema as a view on that table.
It could also support multiple views of the same table, e.g. to satisfy specific per-instrument requirements.
This approach could minimize the need for schema migrations of the actual table, replacing them with the re-definition of the views.


Design decisions
================

The long running time for the ``find-first`` queries is certainly a blocker for view based implementations.
Still those solutions could be considered if query time can be drastically reduced.
Online view is probably the least favorite solution because of lack of indexing which can harm the performance of ObsCore queries.

In general, solutions based on server-side implementation, either in plain SQL or PL/Python, has potential for duplicating some of the ``daf_butler`` complex logic.
Proving or testing that duplicated logic is implemented correctly may be a major issue.
The need to transform spatial regions from ``sphgeom`` encoding to ADQL also makes SQL-based solutions problematic, although this issue can be mitigated by replacing ``sphgeom`` binary encoding with some other text-based representation in the Registry schema.

Ideas for client-side implementation
------------------------------------

Overall, a solution based on client-side updates may have fewer potential issues compared to other solutions.
Implementation of this approach can be straightforward, though the concerns related to collection management also need a consideration.
Like other parts of the Registry, the ObsCore-related operations can be implemented via a separate manager class, which can be optional for repositories that do not need ObsCore support.

There are several Registry operations that may need to update ObsCore table:

- bulk import of the exported data, or ingest of raw datasets;
- some regular ``Butler.put()`` inserts;
- addition of a non-empty run collection to a chained collection if chained collection is mirrored to the ObsCore table;
- removal of individual datasets or a whole run collection;
- removal of a run collection from a chained collection, keeping run collection.

These operation could call methods of a new ObsCore manager instance and pass the information necessary for updating ObsCore table.
Addition of the new datasets should be straightforward, minor concern is potential performance penalty to query the Registry for additional information that needs to be stored with the new dataset.
Complete removal of a dataset from the Registry should also cause its removal from ObsCore table, this could be handled transparently via the foreign key on ``dataset`` table and CASCADE action.
Changes in collections composition may need some attention but should not be hard to implement.

The schema of the ObsCore table is likely to change over time.
The configuration which controls the ObsCore export process determines both the schema of the table and its contents.
To support changing configuration it should be possible to utilize the same migration tools that are used by other Registry managers.
One potentially interesting issue for this approach is whether one Registry could have more than one ObsCore table based on different configurations.

Collection issues with client-side updates
------------------------------------------

Similarly to the standalone script in ``dax_obscore``, a new ObsCore manager should be configured with a list of collections and dataset types to monitor for new datasets.
Allowing arbitrary types of collections in the collection list causes significant issues with consistency and performance, few examples include:

- Changes in the composition of monitored chained collections can result in addition of non-empty run collections.
  This will cause an insertion of many records in obscore table.
- Mixture of run, tagged, and calibration collections results in potentially multiple origins of a single obscore record.
  Removal of records from obscore table (e.g. due to its being disassociated from a tagged collection) would need checks of all other origins for records existence.
- Bulk insertion or additional checks during removal of obscore records pose additional complications for regular Registry database operation.
  They can potentially be performed by a separate standalone "cleanup" script which can run regularly, but that script cannot scale reasonably with growing number of records.
- It is similarly difficult to make contents of the obscore table consistent with the results of ``Registry.queryDatasets()`` method with ``findFirst=True``.
  Supporting this feature would need checks that inserted dataset can potentially replace dataset present in later collections, or should be ignored if a matching dataset already exists in previous collections.

Some of these issues can be resolved by limiting the kind of collections that can appear in the list.
Two possible complementary options include multiple run collections, or a single tagged collection.

If ObsCore manager is configured with multiple run collection, excluding tagged and calibration collections, then the issue with multiple possible origins of obscore records does not exist (single dataset can only exist in a single run collection).
Run collections can be configured as a list of exact collection names or a patterns/regular expressions matching multiple collections.
Chained collections cannot be used for configuration as it is possible to add a non-empty run collection to a chained collection, which would trigger a bulk insert.
Multiple run collections do not solve issue with ``Registry.queryDatasets()`` and ``findFirst=True``, it would be a significant complication and performance issue to try to query other collections while inserting datasets into obscore table.

Alternative approach is to shift hard decisions to some external logic by only monitoring a single tagged collection.
Client code would be required to associate datasets that should appear in obscore table with a special tagged collection.
This also solves issue with ``findFirst=True``, because single tagged collection can only contain unique dataset type and DataId combination.
Associating a dataset with a special obscore collection would potentially require disassociation of a matching existing dataset that should be replaced.

An implementation of ObsCore manager could support both types of collections, one specific option would have to be selected during registry configuration.
A configuration of ObsCore manager is persisted in the registry,.
Applying any changes to this configuration, including changes to the list of collections, or change of collection types, requires schema/data migration with potential downtime.


.. _dax_obscore: https://github.com/lsst-dm/dax_obscore
.. _incremental vew update: https://wiki.postgresql.org/wiki/Incremental_View_Maintenance
.. _PL/Python: https://www.postgresql.org/docs/14/plpython.html
