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
there for DC3b. These tests are documented at
https://dev.lsstcorp.org/trac/wiki/db/SpatialJoinPerf. We found that
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

We timed these tests. This test is described at
https://dev.lsstcorp.org/trac/wiki/db/BuildSubPart. These tests showed
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
https://dev.lsstcorp.org/trac/wiki/db/SubPartOverhead. The on-the-fly
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
largest table, ForcedSource, had nearly 4 billion rows.\ [#]_
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
spatial constraints had to be optimized.\ [#]_ The query
optimizations have been documented at
https://dev.lsstcorp.org/trac/wiki/db/MySQL/Optimizations.

.. [#] It is worth noting that in real production we do not anticipate
   to manage billion+ rows in a *single physical table* - the Qserv system
   that we are developing will split every large table into smaller,
   manageable pieces.

.. [#] Some of these optimizations will not be required when we use
   Qserv, as Qserv will apply them internally.

References
----------

.. bibliography:: bibliography.bib
  :encoding: latex+latin
  :style: lsst_aa

.. note::

  This document was originally published as a part of :cite:`Document-11625` and then part of :cite:`LDM-135`.
