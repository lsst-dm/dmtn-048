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
     :target: http://target.link/url

     Caption text.

   Run: ``make html`` and ``open _build/html/index.html`` to preview your work.
   See the README at https://github.com/lsst-sqre/lsst-technote-bootstrap or
   this repo's README for more info.

   Feel free to delete this instructional comment.

:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. Add content below. Do not include the document title.

Design Trade-Offs
=================

The LSST database design involves many architectural choices. Example of
architectural decisions we faced include how to partition the tables,
how many levels of partitioning is needed, where to use an index, how to
normalize the tables, or how to support joins of the largest tables.
This chapter covers the test we run to determine the optimal
architecture of MySQL-based system.

.. _standalone-tests:

Standalone Tests
----------------

.. _spatial-join-performance:

Spatial join performance
~~~~~~~~~~~~~~~~~~~~~~~~

This test was run to determine how quickly we can do a spatial self-join
(find objects within certain spatial distance of other objects) inside a
single table. Ultimately, in our architecture, a single table represents
a single partition (or sup-partition). The test involved trying various
options and optimizations such as using different indexes (clustered and
non clustered), precalculating various values (like ``COS(RADIANS(decl))``),
and reordering predicates. We run these tests for all reasonable table
sizes (using MySQL and PostgreSQL). We measured CPU and disk I/O to
estimate impact on hardware. In addition, we re-run these tests on the
lsst10 machine at NCSA to understand what performance we may expect
there for DC3b. These tests are documented :ref:`below <spatial-join-performance-trac>`.
We found that
PostgreSQL was 3.7x slower for spatial joins over a range of row counts,
and reducing the row-count per partition to less than 5k rows was
crucial in achieving lowering compute intensity, but that predicate
selectivity could compensate for a 2-4x greater row count.

.. _building-sub-partitions:

Building sub-partitions
~~~~~~~~~~~~~~~~~~~~~~~

Based on the “spatial join performance” test we determined that in order
to speed up self-joins within individual tables (partitions), these
partitions need to be very small, :math:`O(\mathrm{few~} K)` rows.
However, if we partition large tables into a very large number of small
tables, this will result in unmanageable number of tables (files). So,
we determined we need a second level of partitioning, which we call
*sub-partition on the fly*. This test included:

- sub-partitioning through queries:

  1. one query to generate one sub-partition

  2. relying on specially introduced column (``subPartitionId``).

- segregating data into sub-partitions in a client C++ program,
  including using a binary protocol.

We timed these tests. This test is described :ref:`below <building-sub-partitions-trac>`.
These tests showed
that it was fastest to build the on-the-fly sub-partitions using SQL in
the engine, rather than performing the task externally and loading the
sub-partitions back into the engine.

.. _sub-partition-overhead:

Sub-partition overhead
~~~~~~~~~~~~~~~~~~~~~~

We also run detailed tests to determine overhead of introducing
sub-partitions. For this test we used a 1 million row table, measured
cost of a full table scan of such table, and compared it against
scanning through a corresponding data set partitioned into
sub-partitioned. The tests involved comparing in-memory with
disk-based tables. We also tested the influence of introducing
“skinny” tables, as well as running sub-partitioning in a client C++
program, and inside a stored procedure. These tests are described at
:ref:`below <sub-partitioning-overhead-trac>`. The on-the-fly
overhead was measured to be 18% for ``select *`` queries, but
3600% if only one column (the skinniest selection) was needed.

.. _avoiding-materializing-sub-partitions:

Avoiding materializing sub-partitions
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

We tried to run near neighbor query on a 1 million row table. A starting
point is 1000 sec which is ~16 min 40 sec (based on earlier tests we
determined it takes 1 sec to do near neighbor for 1K row table).

The testing included:

- Running near neighbor query by selecting rows with given subChunkId
  into in memory table and running near neighbor query there. It took 7
  min 43 sec.

- Running near neighbor query by running neighbor once for each
  subChunkId, without building sub-chunks. It took 39 min 29 sec.

- Running near neighbor query by mini-near neighbor once for each
  subChunkId, without building sub-chunks, using in-memory table. It
  took 13 min 13 sec.

.. _billion-row-table:

Billion row table / reference catalog
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

One of the catalogs we will need to support is the reference catalog,
even in DC3b it is expected to contain about one billion rows. We have
run tests with a table containing 1 billion rows catalog (containing
USNO-B data) to determine how feasible it is to manage a billion row
table without partitioning it. These tests are described in details at:
https://dev.lsstcorp.org/trac/wiki/DbStoringRefCat This revealed that
single 1-billion row table usage is adequate in loading and indexing,
but query performance was only acceptable when the query predicates
selectivity using an index was a small absolute number of rows (1%
selectivity is too loose). Thus a large fraction of index-scans were
unacceptably slow and the table join speed was also slow.

.. _compression:

Compression
~~~~~~~~~~~

We have done extensive tests to determine whether it is cost effective
to compress LSST databases. This included measuring how different data
types and indexes compress, and performance of compressing and
decompressing data. These tests are described in details at
https://dev.lsstcorp.org/trac/wiki/MyIsamCompression. We found that
table data compressed only 50%, but since indexes were not compressed,
there was only about 20% space savings. Table scans are significantly
slower due to CPU expense, but short, indexed queries were only impacted
40-50%.

.. _full-table-scan-performance:

Full table scan performance
~~~~~~~~~~~~~~~~~~~~~~~~~~~

To determine performance of full table scan, we measured:

1. raw disk speed with ``dd if=<large file> of=/dev/zero`` and got
   54.7 MB/sec (2,048,000,000 bytes read in 35.71 sec)

2. speed of ``select count(*) from XX where muRA = 4.3`` using a 1
   billion row table. There was no index on muRA, so this forced a full
   table scan. Note that we did not do ``SELECT *`` to avoid measuring
   speed of converting attributes. The scan of 72,117,127,716 bytes took
   28:49.82 sec, which is 39.8 MB/sec.

So, based on this test the full table scan can be done at *73% of the
raw disk speed* (using MySQL MyISAM).

.. _low-volume-queries:

Low-volume queries
~~~~~~~~~~~~~~~~~~

A typical low-volume queries to the best of our knowledge can be divided
into two types:

- analysis of a single object. This typically involves locating a small
  number of objects (typically just one) with given objectIds, for
  example find object with given id, select attributes of a given
  galaxy, extract time series for a given star, or select variable
  objects near known galaxy. Corresponding representative queries:

  .. code:: sql

     SELECT * from Object where objectId=<xx>
     SELECT * from Source where objectId =<xx>

- analysis of objects meeting certain criteria in a small spatial
  region. This can be represented by a query that selects objects in a
  given small ra/dec bounding box, so e.g.:

  .. code:: sql

     SELECT * FROM Object
     WHERE ra BETWEEN :raMin AND :raMax
     AND decl BETWEEN :declMin AND :declMax
     AND zMag BETWEEN :zMin AND :zMax

Each such query will typically touch one or a few partitions (few if the
needed area is near partition edge). In this test we measured speed for
a single partition.

Proposed partitioning scheme will involve partitioning each large table
into a “reasonable” number of partitions, typically measured in low tens
of thousands. Details analysis are done in the storage spreadsheet
:cite:`LDM-141`. Should we need to, we can partition
the largest tables into larger number of smaller partitions, which would
reduce partition size. Given the hardware available and our time
constraints, so far we have run tests with up to 10 million row
partition size.

We determined that if we use our custom spatial index (“subChunkId”), we
can extract 10K rows out of a 10 million row table in 30 sec. This is
too long – low volume queries require under 10 sec response time.
However, if we re-sort the table based on our spatial index, that same
query will finish in under 0.33 sec.

We expect to have 50 low volume queries running at any given time. Based
on details disk I/O estimates, we expect to have ~200 disk spindles
available in DR1, many more later. Thus, it is likely majority of low
volume queries will end up having a dedicated disk spindle, and for
these that will end up sharing the same disk, caching will likely help.

Note that these tests were done on fairly old hardware (7 year old).

In summary, we demonstrated low-volume queries can be answered through
an index (objectId or spatial) in well under 10 sec.

.. _ssd:

Solid state disks
~~~~~~~~~~~~~~~~~

We also run a series of tests with solid state disks to determine where
it would be most cost-efficient to use solid state disks. The tests are
described in details in :cite:`Document-11701`. We found that concurrent query execution
is dominated by software inefficiencies when solid-state devices (SSDs)
with fast random I/O are substituted for slow disks. Because the cost
per byte is higher for SSDs, spinning disks are cheaper for bulk
storage, as long as access is mostly sequential (which can be
facilitated with shared scanning). However, because the cost per random
I/O is much lower for SSDs than for spinning disks, using SSDs for
serving indexes, exposure metadata, perhaps even the entire Object
catalog, as well as perhaps for temporary storage is advised. This is
true for the price/performance points of today's SSDs. Yet even with
high IOPS performance from SSDs, table-scan based selection is often
faster than index-based selection: a table-scan is faster than an index
scan when >9% of rows are selected (cutoff is >1% for spinning disk).
The commonly used 30% cutoff does not apply for large tables for present
storage technology.

.. _data-challenge-tests:

Data Challenge Related Tests
----------------------------

During each data challenge we test some aspects of database performance
and/or scalability. In DC1 we demonstrated ingest into database at the
level of 10% of DR1, in DC2 we demonstrated near-real-time object
association, DC3 is demonstrating catalog construction and DC4 will
demonstrate the end user query/L3 data production.

In addition to DC-related tests, we are running standalone tests,
described in detail in :cite:`DMTR-12`, :cite:`DMTR-21` and :cite:`LDM-552`.

.. _dc1:

DC1: data ingest
~~~~~~~~~~~~~~~~

We ran detailed tests to determine data ingest performance. The test
included comparing ingest speed of MySQL against SQL Server speed, and
testing different ways of inserting data to MySQL, including direct
ingest through INSERT INTO query, loading data from ASCII CSV files. In
both cases we tried different storage engines, including MyISAM and
InnoDB. Through these tests we determined the overhead introduced by
MySQL is small (acceptable). Building indexes for large tables is slow,
and requires making a full copy of the involved table. These tests are
described in details in Docushare Document-1386. We found that as long
as indexes are disabled during loading, ingest speed is typically CPU
bound due to data conversion from ASCII to binary format. We also found
that ingest into InnoDB is usually ~3x slower than into MyISAM,
independently of table size.

.. _dc2:

DC2: source/object association
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

One of the requirements is to associated DiaSource with Object is almost
real-time. Detailed study how to achieve that has been done in
conjunction with the Data Challenge 2. The details are covered at:
https://dev.lsstcorp.org/trac/wiki/db/DC2/PartitioningTests and the
pages linked from there. We determined that we need to maintain a narrow
subset of the data, and fetch it from disk to memory right before the
time-critical association in order to minimize database-related delays.

.. _dc3:

DC3: catalog construction
~~~~~~~~~~~~~~~~~~~~~~~~~

In DC3 we demonstrated catalog creation as part of the Data Release
Production.

.. _winter2013-querying:

Winter-2013 Data Challenge: querying database for forced photometry
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Prior to running Winter-2013 Data Challenge, we tested performance of
MySQL to determine whether the database will be able to keep up with
forced photometry production which runs in parallel. We determined that
a single MySQL server is able to easily handle 100-200 simultaneous
requests in well under a second. As a result we chose to rely on MySQL
to supply input data for forced photometry production. Running the
production showed it was the right decision, e.g., the database
performance did not cause any problems. The test is documented at
https://dev.lsstcorp.org/trac/wiki/db/tests/ForcedPhoto.

.. _winter2013-partitioning:

Winter-2013 Data Challenge: partitioning 2.6 TB table for Qserv
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The official Winter-2013 production database, as all past data
challenged did not rely on Qserv, instead, plain MySQL was used instead.
However, as an exercise we partitioned and loaded this data set into
Qserv. This data set relies on table views, so extending the
administrative tools and adding support for views inside Qserv was
necessary. In the process, administrative tools were improved to
flexibly use arbitrary number of batch machines for partitioning and
loading the data. Further, we added support for partitioning RefMatch\*
tables; RefMatch objects and sources have to be partitioned in a unique
way to ensure they join properly with the corresponding Object and
Source tables.

.. _winter2014-multi-billion-row:

Winter-2013 Data Challenge: multi-billion-row table
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The early Winter 2013 production resulted in 2.6 TB database; the
largest table, ForcedSource, had nearly 4 billion rows.\ [*]_
Dealing with multi-billion row table is non-trivial are requires special
handling and optimizations. Some operations, such as building an index
tend to take a long time (tens of hours), and a single ill-tuned
variable can result in 10x (or worse) performance degradation. Producing
the final data set in several batches was in particular challenging, as
we had to rebuild indexes after inserting data from each batch. Key
lessons learned have been documented at
https://dev.lsstcorp.org/trac/wiki/mysqlLargeTables. Issues we uncovered
with MySQL (myisamchk) had been reported to the MySQL developers, and
were fixed immediately fixed.

In addition, some of the more complex queries, in particular these with
spatial constraints had to be optimized.\ [*]_ The query
optimizations have been documented at
https://dev.lsstcorp.org/trac/wiki/db/MySQL/Optimizations.

.. _other-demonstrations:

Other Demonstrations
====================

.. _demo-shared-scans:

Shared Scans
------------

We have conducted preliminary empirical evaluation of our basic shared
scan implementation. The software worked exactly as expected, and we
have not discovered any unforeseen challenges. For the tests we used a
mix of queries with a variety of filters, different CPU load, different
result sizes, some with grouping, some with aggregations, some with
complex math. Specifically, we have measured the following:

- A single full table scan through the Object table took ~3 minutes.
  Running a mix 30 such queries using our shared scan code took 5 min 27
  sec (instead of expected ~1.5 hour it’d take if we didn’t use the
  shared scan code.)

- A single full table scan through Source table took between ~14 min 26
  sec and 14 min 36 sec depending on query complexity. Running a mix of
  30 such queries using shares scan code took 25 min 30 sec.  (instead
  of over 7 hours).

In both cases the extra time it took comparing to the timing of a single
query was related to (expected) CPU contention: we have run 30
simultaneous queries on a slow, 4-core machine.

In addition, we demonstrated running simultaneously a shared scan plus
short, interactive queries. The interactive queries completed as
expected, in some cases with a small (1-2 sec) delay.

.. _demo-fault-tolerance:

Fault Tolerance
---------------

To prove Qserv can gracefully handle faults, we artificially triggered
different error conditions, such as corrupting random parts of a
internal MySQL files while Qserv is reading them, or corrupting data
sent between various components of the Qserv (e.g., from the `XRootD`_ to
the master process).

.. _demo-worker-failure:

Worker failure
~~~~~~~~~~~~~~

These tests are meant to simulate worker failure in general, including
spontaneous termination of a worker process and/or inability to
communicate with a worker node.

When a relevant worker (i.e. one managing relevant data) has failed
prior to query execution, either 1) duplicate data exists on another
worker node, in which case `XRootD`_ silently routes requests from the
master to this other node, or 2) the data is unavailable elsewhere, in
which case `XRootD`_ returns an error code in response to the master's
request to open for write. The former scenario has been successfully
demonstrated during multi-node cluster tests. In the latter scenario,
Qserv gracefully terminates the query and returns an error to the user.
The error handling of the latter scenario involves recently developed
logic and has been successfully demonstrated on a single-node,
multi-worker process setup.

Worker failure during query execution can, in principle, have several
manifestations.

1. If `XRootD`_ returns an error to the Qserv master in response to a
   request to open for write, Qserv will repeat request for open a fixed
   number (e.g. 5) of times. This has been demonstrated.

2. If `XRootD`_ returns an error to the Qserv master in response to a
   write, Qserv immediately terminates the query gracefully and returns
   an error to the user. This has been demonstrated. Note that this may
   be considered acceptable behavior (as opposed to attempting to
   recover from the error) since it is an unlikely failure-mode.

3. If `XRootD`_ returns an error to the Qserv master in response to a
   request to open for read, Qserv will attempt to recover by
   re-initializing the associated chunk query in preparation for a
   subsequent write. This is considered the most likely manifestation of
   worker failure and has been successfully demonstrated on a
   single-node, multi-worker process setup.

4. If `XRootD`_ returns an error to the Qserv master in response to a read,
   Qserv immediately terminates the query gracefully and returns an
   error to the user. This has been demonstrated. Note that this may be
   considered acceptable behavior (as opposed to attempting to recover
   from the error) since it is an unlikely failure-mode.

.. _demo-data-corruption:

Data corruption
~~~~~~~~~~~~~~~

These tests are meant to simulate data corruption that might occur on
disk, during disk I/O, or during communication over the network. We
simulate these scenarios in one of two ways. 1) Truncate data read via
`XRootD`_ by the Qserv master to an arbitrary length. 2) Randomly choose a
single byte within a data stream read via `XRootD`_ and change it to a
random value. The first test necessarily triggers an exception within
Qserv. Qserv responds by gracefully terminating the query and returning
an error message to the user indicating the point of failure (e.g.
failed while merging query results). The second test intermittently
triggers an exception depending on which portion of the query result is
corrupted. This is to be expected since Qserv verifies the format but
not the content of query results. Importantly, for all tests, regardless
of which portion of the query result was corrupted, the error was
isolated to the present query and Qserv remained stable.

.. _demo-future-tests:

Future tests
~~~~~~~~~~~~

Much of the Qserv-specific fault tolerance logic was recently developed
and requires additional testing. In particular, all worker failure
simulations described above must be replicated within a multi-cluster
setup.

.. _multiple-qserve:

Multiple Qserv Installations on a Single Machine
------------------------------------------------

Once in operations, it will be important to allow multiple qserv
instances to coexist on a single machine. This will be necessary when
deploying new Data Release, or for testing new version of the software
(e.g., MySQL, or Qserv). In the short term, it is useful for shared code
development and testing on a limited number of development machines we
have access to. We have successfully demonstrated Qserv have no
architectural issues or hardcoded values such as ports or paths that
would prevent us from running multiple instances on a single machine.

.. _spatial-join-performance-trac:

Spatial Join Performance
========================

This section describes performance of spatial join queries as of early 2010.\ [*]_

In practice, we expect spatial joins on Object table only. To avoid the
join on multi-billion row table, the table will be partitioned
(chunked), as described in :cite:`Document-26276`.
The queries discussed here correspond to a query executed
on a single chunk; e.g., table partitioning and distribution of
partitions across disks or nodes are not considered here. It is expected
many such chunk-based queries will execute in parallel.

Used schema:

.. code:: sql

    CREATE TABLE X (
      objectId  BIGINT NOT NULL PRIMARY KEY,
      ra        FLOAT NOT NULL,
      decl      FLOAT NOT NULL,
      muRA      FLOAT NOT NULL,
      muRAErr   FLOAT NOT NULL,
      muDecl    FLOAT NOT NULL,
      muDeclErr FLOAT NOT NULL,
      epoch     FLOAT NOT NULL,
      bMag      FLOAT NOT NULL,
      bFlag     FLOAT NOT NULL,
      rMag      FLOAT NOT NULL,
      rFlag     FLOAT NOT NULL,
      b2Mag     FLOAT NOT NULL,
      b2Flag    FLOAT NOT NULL,
      r2Mag     FLOAT NOT NULL,
      r2Flag    FLOAT NOT NULL
    )

Used data: USNO catalog.

A first version of the query:

.. code:: sql

    SELECT count(*)
    FROM   X o1, X o2
    WHERE  o1.objectId <> o2.objectId
      AND  ABS(o1.ra - o2.ra) < 0.00083 / COS(RADIANS(o2.decl))
      AND ABS(o1.decl - o2.decl) < 0.00083

Precalculating COS(RADIANS(decl)):

.. code:: sql

      ALTER TABLE X ADD COLUMN cosRadDecl FLOAT
      UPDATE X set cosRadDecl = COS(RADIANS(decl))

and changing the order of predicates as follows:

.. code:: sql

    SELECT count(*)
    FROM   X o1, X o2
    WHERE  ABS(o1.ra - o2.ra) < 0.00083 / o2.cosRadDecl
      AND  ABS(o1.decl - o2.decl) < 0.00083
      AND  o1.objectId <> o2.objectId

improves the execution time by 36% for mysql, and 38% for postgres.

Here is the timing for this (optimized) query.

+---------+---------+------------+
| nRows   | mysql   | postgres   |
+---------+---------+------------+
| [K]     | [sec]   | [sec]      |
+---------+---------+------------+
| 1       | 1       | 5          |
+---------+---------+------------+
| 2       | 5       | 19         |
+---------+---------+------------+
| 3       | 11      | 40         |
+---------+---------+------------+
| 4       | 18      | 69         |
+---------+---------+------------+
| 5       | 28      | 103        |
+---------+---------+------------+
| 10      | 101     | 371        |
+---------+---------+------------+
| 15      | 215     | 797        |
+---------+---------+------------+
| 20      | 368     |            |
+---------+---------+------------+
| 25      | 566     |            |
+---------+---------+------------+

Postgres is ~3.7x slower than mysql in this case. It is probably
possible to tune postgreSQL a little, but is it unlikely it will match
MySQL performance.

Each of these queries for both mysql and postgres were completely
CPU-dominated. The test was executed on an old-ish Sun 1503 MHz sparcv9
processor.

Also, see: :cite:`Document-11701` - the test rerun on a fast machine (dash-io).

Using index
-----------

The near neighbor query can be further optimized by introducing an
index. Based on the tests we run with mysql, in order to force mysql to
use an index for this particular query, we have to build a composite
index on all used columns, or build a composite index on (ra, decl,
cosRadDecl) and remove o1.objectId <> o2.objectId predicate (this
predicate would have to be applied in a separate step). The timing for
the query with index and without the objectId comparison:

+---------+---------+---------+----------+
| nRows   | was     | now     | faster   |
+---------+---------+---------+----------+
| [K]     | [sec]   | [sec]   | [%       |
+---------+---------+---------+----------+
| 5       | 28      | 16.7    | 40       |
+---------+---------+---------+----------+
| 10      | 101     | 67.2    | 33       |
+---------+---------+---------+----------+
| 15      | 215     | 150.8   | 30       |
+---------+---------+---------+----------+
| 20      | 368     | 269.5   | 27       |
+---------+---------+---------+----------+
| 25      | 566     | 420.3   | 26       |
+---------+---------+---------+----------+

The speedup from using an index will likely be bigger for wider tables.

CPU utilization
---------------

How many CPUs do we need to do full correlation on 1 billion row table
(DC3b)?

+-------------+--------------+---------------+--------------+--------------------+
| # chunks    | rows/chunk   | seconds per   | total        | total core-hours   |
+-------------+--------------+---------------+--------------+--------------------+
|             |              | self-join     | core-hours   | if 16 cores used   |
+-------------+--------------+---------------+--------------+--------------------+
|             |              | in 1 chunk    | needed       | twice faster       |
+-------------+--------------+---------------+--------------+--------------------+
| 40,000      | 25,000       | 566           | 6,289        | 196                |
+-------------+--------------+---------------+--------------+--------------------+
| 50,000      | 20,000       | 368           | 5,111        | 160                |
+-------------+--------------+---------------+--------------+--------------------+
| 66,666      | 15,000       | 215           | 3,981        | 124                |
+-------------+--------------+---------------+--------------+--------------------+
| 100,000     | 10,000       | 101           | 2,806        | 88                 |
+-------------+--------------+---------------+--------------+--------------------+
| 200,000     | 5,000        | 28            | 1,556        | 49                 |
+-------------+--------------+---------------+--------------+--------------------+
| 250,000     | 4,000        | 18            | 1,250        | 39                 |
+-------------+--------------+---------------+--------------+--------------------+
| 333,333     | 3,000        | 11            | 1,019        | 31                 |
+-------------+--------------+---------------+--------------+--------------------+
| 500,000     | 2,000        | 5             | 694          | 22                 |
+-------------+--------------+---------------+--------------+--------------------+
| 1,000,000   | 1,000        | 1             | 278          |                    |
+-------------+--------------+---------------+--------------+--------------------+

Realistically, we can count on ~2 8-core servers, ~twice faster than the
CPUs used in these tests. That means 1 million chunk version would
finish in 9 hours, 66K chunk version would need 5 days to finish.

Near neighbor without building sub-chunking
-------------------------------------------

We tested the performance of running nn query without explicitly
building sub-chunking. In all these tests describe here we tried to run
nn on a 1 million row table. A starting point is 1000 sec (1 sec per 1K
rows), which is ~16 min 40 sec.

First test: running near neighbor query by selecting rows with given
subChunkId into in memory table and running nn query there. This test is
`here </_static/test001.py>`__. It took
7 min 43 sec.

Second test: running near neighbor query by running neighbor once for
each subChunkId, without building sub-chunks. This test is `here </_static/test002.py>`__. It took
39 min29 sec.

Third test: runnig near neighbor query by mini-near neighbor once for
each subChunkId, without building sub-chunks, using in-memory table.
This test is `here </_static/test003.py>`__. It took 13 min 13 sec.

Near neighbor with predicates
-----------------------------

Note that full n^2 correlation without any cuts is the worst possible
spatial join, rarely needed in practice. A most useful form of near
neighbour search is a correlation with some predicates. Here is an
example hypothetical (non-scientific) query, written for the data set
used in our tests:

.. code:: sql

    SELECT count(*)
    FROM   X o1, X o2
    WHERE  o1.bMag BETWEEN 20 AND 20.13
      AND  o2.rMag BETWEEN 19.97 AND 20.15
      AND  ABS(o1.ra - o2.ra) < 0.00083 / o2.cosRadDecl
      AND  ABS(o1.decl - o2.decl) < 0.00083
      AND  o1.objectId <> o2.objectId

For the data used, the applied o1.bMag cut selects ~2% of the table, so
does the o2.rMag cut (if applied independently).

With these cuts, the near neighbour query on 25K-row table takes 0.3 sec
in mysql and 1 sec in postgres (it'd probably run even faster with
indexes on bMag and rMag). So if we used mysql we would need only 3
(1503 MHz) CPUs do run query over 1 billion rows in one hour.

For selectivity 20% it takes mysql 12 sec to finish and postgres needs
35 sec. In this case mysql would need 133 (1503 MHz) CPUs to run this
query over 1 billion rows in one hour.

This clearly shows predicate selectivity is one of the most important
factors determining how slow/fast the spatial queries will run. In
practice, if the selectivity is <10%, chunk size = ~25K or 50K rows
should work well.

``[perhaps we need to do more detailed study regarding predicate selectivity]``

Numbers for lsst10
------------------

As of late 2009 the lsst10 server had 8 cores.

Elapsed time for a single job is comparable to elapsed time of 8 jobs
run in parallel (difference is within 2-3%, except for very small
partitions, where it reaches 10%).

Testing involved running 8 “jobs”, where each job was a set of queries
executed sequentially. Each query was a near neighbor query:

.. code:: sql

    SELECT count(*) AS neighbors
    FROM   XX o1 FORCE INDEX (idxRDC),
           XX o2 FORCE INDEX (idxRDC)
    WHERE  ABS(o1.decl - o2.decl) < 0.00083
      AND  ABS(o1.ra - o2.ra) < 0.00083 / o2.cosRadDecl

    -- idxRDC was defined as "ADD INDEX idxRDC(ra, decl, cosRadDecl)"

Each query run on a single partition. Results:

+----------------------+--------------------------------------+----------------------------------+------------------------------------+
| rows per partition   | seconds to self-join one partition   | rows processed per elapsed sec   | time to process 150m rows (DC3b)   |
+----------------------+--------------------------------------+----------------------------------+------------------------------------+
| 0.5K                 | 0.05                                 | 80K                              | 31min                              |
+----------------------+--------------------------------------+----------------------------------+------------------------------------+
| 1K                   | 0.16                                 | 50K                              | 50min                              |
+----------------------+--------------------------------------+----------------------------------+------------------------------------+
| 2K                   | 0.59                                 | 27K                              | 1h 32min                           |
+----------------------+--------------------------------------+----------------------------------+------------------------------------+
| 5K                   | 3.63                                 | 11K                              | 3h 47min                           |
+----------------------+--------------------------------------+----------------------------------+------------------------------------+
| 10K                  | 14.68                                | 5.5K                             | 7h 39min                           |
+----------------------+--------------------------------------+----------------------------------+------------------------------------+
| 15K                  | 40.09                                | 3K                               | 14h                                |
+----------------------+--------------------------------------+----------------------------------+------------------------------------+

See `200909nearNeigh-lsst10.xls </_static/200909nearNeigh-lsst10.xls>`__ for details.

.. _building-sub-partitions-trac:

Building sub-partitions
=======================

We tested cost of building sub partitions on the fly.\ [*]_ This task involves
taking a large partition and segregating rows into different tables,
such that all rows with a given subChunkId end up in the same table.

Numbers for DC3b
----------------

In DC3b we will have ~150 million rows in the Object table.

A reasonable partitioning/sub-partitioning scheme:

-  1,500 partitions, 100K rows per partition.
-  a partition dynamically split into 100 sub-partitions 1K row each.

Justification
~~~~~~~~~~~~~

-  we want to keep the number of partitions <30K. Each partition = table
   = 3 files. 200K partitions would = 2GB of internal structure to be
   managed by xrootd (in memory).
-  we want relatively small sub-partitions (<2K rows) in order for near
   neighbor query to run fast, see the :ref:`analysis above <spatial-join-performance-trac>`.
-  sub-partitions can't be too small because the overlap will start
   playing big role, eg overlap over ~20% starts to become unacceptable.

Object density will not vary too much, and will be ~ 5e5 / deg^2, or 140
objects / sqArcmin. (star:galaxy ratio will vary, but once we add both
together it wont because the regions that have very high star densities
also have very high extinction, so that background galaxies are very
difficult to detect)

So a 1K-row subpartition will roughly cover 3x3 sq arcmin. Given SDSS
precalculated neighbors for 0.5 arcmin distance, this looks reasonable
(eg, the overlap is ~ 17% for the largest distance searched.)

Testing
-------

So, based on the above, we tested performance of splitting a single 100k
row table into a 100 1k tables.

Test 1: “CREATE TABLE SELECT FROM WHERE” approach
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The simplest approach is to run in the loop

.. code:: sql

  CREATE TABLE sp_xxx ENGINE=MEMORY SELECT * FROM XX100k where subChunkId=xxx;

for each sub chunk, where xxx is a subChunkId.

Timing: ~ 1.5 sec

Test 2: Segregating in a client program (ascii)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Second, we created a C++ program that does the following:

-  ``SELECT * from XX100k`` (reads the whole 100k row partition into
   memory)
-  segregate in memory rows based on subChunkId value
-  buffer them (a buffer per subChunkId)
-  flush each buffer to a table using ``INSERT INTO sp_xxx VALUES
   (1,2,3), (4,3,4), ..., (3,4,5)``

Note that the values are sent from mysqld to the client as ascii, and
sent back from the client back to the server as ascii too.

The client job was run on the same machine where the mysqld run.

Timing: ~3x longer than the first test. Only ~2-3% of the total elapsed
time was spent in the client code, the rest was waiting for mysqld, so
optimizing the client code won't help.

In summary, it is worse than the first test.

Test 3: Segregating in a client (binary protocol)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

We run a test similar to the previous one, but to avoid the overhead of
converting rows to ascii we used the binary protocol. In practice,

-  reading from the input table was done through a prepared statement
-  writing to a subChunk tables was done through prepared statements
   ``INSERT INTO x VALUES (?,?,?), (?,?,?), ..., (?,?,?)``, and values
   were appropriately bound.

Note that a prep statement had to be created for each output table
(table names can't parameterized).

Relevant code is `here </_static/binaryProtocol.zip>`__.

Timing: ~2x longer than the first test. Similarly to the previous test,
only 2-3% of time was spent in the client code.

Using ``CLIENT_COMPRESS`` flag makes things ~2x worse.

In summary, it is better then the previous test, but still worse than
the first test.

Note on concurrency
-------------------

So the test 1 is the fastest. It creates ~66 tables/second.
Unfortunately, running 8 of these jobs in parallel (1 per core) on
lsst10 takes ~8-9 sec, not 1.5 as one would expect (presumably due to
mysql metadata contention). Writing from each core to a different
database does not help.

Timing for DC3b
---------------

Assuming we use the “test 1” approach, and assuming we can do 1
partition in 1.5 sec (no speedup from multi-cores), it would take 37.5
min to sub-partition the whole Object table.

Based on :ref:`the analysis above <spatial-join-performance-trac>`, we can do near neighbor join on 1K
row table in 0.16 sec, so we need 16 sec to handle 100 such tables
(equal 1 100k row partition). If we run it in parallel on 7 cores while
the 8th core is sub-partitioning “next” partition, a single 100k-row
partition would (theoretically, we didn't try yet!) be done in 2.3 sec.
That is 57 min for entire DC3b Object catalog, e.g., sub-partitioning
costs us 7 min (near neighbor on 8 cores would finish in 50 min).

Each near-neighbor job does only one read access to the mysql metadata,
so the contention on mysql metadata should not be a problem.

.. _sub-partitioning-overhead-trac:

Sub-partitioning overhead [*]_
==============================

Introduction
------------

As discussed in :cite:`Document-26276`,
it might be handy to be able to split a large chunk of a table on the
fly into smaller sub-zones and sub-chunks.

To determine the overheads associated with this, we run some tests on a
1 million row table which in this case represented a reasonably sized
chunk of a multi-billion row table. The used table had significantly
less columns than the real Object table (72 bytes per row vs expected
2.5K). The test was run directly on the database server. Preparing for
the test involved

1. Created a 1-million row table
2. Precalculated 10 subZoneIds such that each zone had the same
   (or almost the same) number of rows
3. Precalculated 100 subChunkIds (10 per zone) such that each chunk had the
   same (or almost the same) number of rows
4. Created an index on subChunkId. The index selectivity is 1%.

Conclusions
-----------

For ``SELECT *`` type queries, the measured overhead was ~18%. It will
likely be even lower for our wide tables (Object table has 300 columns)

For ``SELECT`` the measured overhead was ~3600% (x36). However

-  if we use shared scans, this cost will be amortized over multiple
   queries. Also, it is very unlikely all queries using a shared scan
   will select one column
-  If it is a non-shared-scan query: we can put into subchunk tables
   only these columns that the query needs plus the primary key.

So in practice, the overhead of building sub-partitions seems
acceptable.

See below how we determined all this.

Test one: determining cost of full table scan
---------------------------------------------

First, to determine the “baseline” (how long it takes to run various
queries that involve a single full table scan) we run the following 4
queries

.. code:: sql

     SELECT * FROM XX                                               -- A
     INSERT INTO memR SELECT * FROM XX                              -- B
     SELECT SUM(bMag+rMag+b2Mag+r2Mag) FROM XX                      -- C
     INSERT INTO memRSum SELECT SUM(bMag+rMag+b2Mag+r2Mag) FROM XX  -- D

Queries A and C represent the cost of querying the data, query B
represents the minimum cost to build sub-chunks, and query D helps to
determine the cost of sending the results to the client.

*memR* and *memRSum* tables are both in-memory.

After running them, we rerun them again to check the effect of potential
caching. The timing for the first set was: 82.27, 10.32, 1.09, 1.06. The
timing for the second set was almost the same: 81.86, 10.48, 1.04, 1.06,
so it looks like caching doesn't distort the results.

The table + indexes used for these tests = ~100MB in size, and they
easily fit into 16GB of RAM we had available. We did not clean up the OS
cache between tests, so it is fair to assume we are skipping ~2 sec
which would be used to fetch the data from disk (which can do 55MB/sec)

Observations: it is expensive to format the results (cost of query B
minus cost of query D = ~9 sec). It is even more expensive to send the
results to the client application (cost of query A minus cost of query B
= ~71 sec)

Test two: determining cost of dealing with many small tables
------------------------------------------------------------

We divided 1 million rows across 100 small tables (10K rows per table).
It took 1.10 sec to run the “SELECT SUM” query sequentially for all 100
tables, so the overhead of dealing with 100 smaller tables instead of
one bigger is negligible.

A similar test with 1000 small tables (1K rows each) took 1.98 sec to
run. Here, the overhead was visibly bigger (close to x2), but it is
still acceptable.

Observation: the overhead of reading from many smaller tables instead of
1 big table is acceptable.

Test three: using subChunkIds
-----------------------------

The test included creating one in-memory table, then for each chunk:

.. code:: sql

      INSERT INTO memT SELECT * FROM XX WHERE subChunkId = <id>
      SELECT SUM(bMag+rMag+b2Mag+r2Mag) FROM memT
      TRUNCATE memT

It took 35.80 sec, and comparing with the “baseline” numbers (queries B
and C) it was about 24 sec slower. Conclusion: “WHERE subChunkId”
introduced 24 sec delay. With no chunking, our SELECT query would
complete in ~1 sec, so the overall overhead is ~x36

Re-clustering the data based on the subChunkId index by doing:

.. code:: sql

    myisamchk <baseDir>/subChunk/XX.MYI --sort-record=3

has minimal effect on the total execution cost. The likely reason is
that the table used for the test easily fits in memory and it is not
fetched from disk.

Doing the same test but with “SELECT \*” instead of
“SUM(bMag+rMag+b2Mag+r2Mag)” took 98.15 sec. Since the baseline query A
took ~83 sec, in this case the overhead was only ~15 sec (18%).

Test four: using “skinny” subChunkIds
-------------------------------------

The test included creating one in-memory table *CREATE TEMPORARY TABLE
memT (magSums FLOAT, objectId BIGINT)*, then for each chunk:

.. code:: sql

      INSERT INTO memT SELECT bMag+rMag+b2Mag+r2Mag, objectId FROM XX WHERE subChunkId = <id>
      SELECT SUM(magSums) FROM memT
      TRUNCATE memT

Essentially instead of blindly copying all columns to subchunks, we are
coping only the minimum that is needed (including objectId which will
always be needed for self-join queries). In this case, the test too
11.39 sec which is over 3x improvement comparing to the previous test.

Test five: sub-partitioning in a client program
-----------------------------------------------

We tried building sub-partitions through a client program (python). The
code was looping through rows returned from 'SELECT \* FROM XX', and for
each row it

-  parsed the row
-  checked subChunkId
-  appended it to appropriate ``INSERT INTO chunk\_ VALUES (...), (...)``
   command

then it run the 'insert' commands.

Doing ``SELECT * FROM <table>``
takes 81 seconds, which is aligned with our baseline numbers (query A).
Then remaining processing took 533 sec. It could probably be optimized a
little, but it is not worth it, it is clear this is not the way to go.

Test six: stored procedure
--------------------------

Finally, we tried sub-partitioning through a stored procedure. Actually,
not the whole sub-partitioning, but just the scanning. The following
stored procedure was used:

.. code:: sql

    CREATE PROCEDURE subPart()
    BEGIN

      DECLARE objId BIGINT;
      DECLARE ra, decl DOUBLE;
      DECLARE muRA, muRAERr, muDecl, muDeclErr, epoch, bMag, bFlag,
              rMag, rFlag, b2Mag, b2Flag, r2Mag, r2Flag FLOAT;
      DECLARE subZoneId, subChunkId SMALLINT;

      DECLARE c CURSOR FOR SELECT * FROM XX;

      OPEN c;

      CURSOR_LOOP: LOOP
        FETCH C INTO objId, ra, decl, muRA, muRAERr, muDecl, muDeclErr, epoch, bMag, bFlag,
                     rMag, rFlag, b2Mag, b2Flag, r2Mag, r2Flag, subZoneId, subChunkId;

      END LOOP;

      CLOSE c;
    END
    //

So, this procedure only scans the data, it doesn't insert rows to the
corresponding sub-tables. It is roughly equivalent to the query B from
the baseline test. This test took 21 seconds, almost the same as the
corresponding baseline query B. Conclusion: pushing this computation to
the stored procedure doesn't visibly improve the performance.


References
==========

.. bibliography:: bibliography.bib
  :encoding: latex+latin
  :style: lsst_aa

.. note::

  This document was originally published as a part of :cite:`Document-11625` and then part of :cite:`LDM-135`.

.. _XRootD: http://xrootd.org

.. [*] It is worth noting that in real production we do not anticipate
   to manage billion+ rows in a *single physical table* - the Qserv system
   that we are developing will split every large table into smaller,
   manageable pieces.

.. [*] Some of these optimizations will not be required when we use
   Qserv, as Qserv will apply them internally.

.. [*] Original location of this report: https://dev.lsstcorp.org/trac/wiki/db/SpatialJoinPerf

.. [*] Original location of this 2009 report: https://dev.lsstcorp.org/trac/wiki/db/BuildSubPart

.. [*] Original location of this 2009 report: https://dev.lsstcorp.org/trac/wiki/db/SubPartOverhead
