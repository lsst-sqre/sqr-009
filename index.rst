..
  Content of technical report.

  See http://docs.lsst.codes/en/latest/development/docs/rst_styleguide.html
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
   See the README at https://github.com/lsst-sqre/lsst-report-bootstrap or
   this repo's README for more info.

   Feel free to delete this instructional comment.

:tocdepth: 1

.. note::
    Work in progress.

Introduction
============

We present the current design and vision for the Science Quality Analysis Harness (SQuaSH).

SQuaSH is a web-based service for tracking Science Pipeline metrics. SQuaSH provides a REST API to which metrics values collected from pipeline tasks can be submitted and stored in a database. It enables DM developers to track the evolution of metric values with time and relate them directly to changes in code or configuration.

The current version of SQuaSH runs at https://squash.lsst.codes, and the reference documentation is available at https://squash.lsst.io.


Design guidelines
=================

The DM Science Data Quality Assurance (SDQA) Conceptual Design [`LDM-522 <http://ls.st/LDM-522>`_] describes the Quality Control (QC) services required to enable the capabilities listed in the LSST Data Quality Assurance plan [`LDM-63 <http://ls.st/LSE-63>`_].

The general guidelines for SQuaSH are the following:

- Implement the concepts developed in the `LSST Verification Framework <https://sqr-019.lsst.io>`_;
- Implement a database to preserve the results of the verification runs, initially from DM Jenkins CI but also other execution environments;
- Enable the visualization of metric values through interactive dashboards;
- Provide notifications/alerts upon metric value deviations from the specifications;
- Enable the drill down to external dashboards and the LSST Science Platform for exploratory analysis.

More recently, the QA Strategy Working Group Report [`DMTN-085 <https://dmtn-085.lsst.io/>`_] compiled a set of recommendations for SQuaSH which are guiding its current development.

Overview
========

.. figure:: _static/overview.png
   :name: SQuaSH overview.
   :target: _static/overview.png

   The figure shows the SQuaSH system and its connection to other DM tools.

Collecting metrics with the LSST Verification Framework
-------------------------------------------------------

The LSST Verification Framework (``lsst.verify``) is the DM framework for collecting metrics from the Science Pipelines. It aims to generalize the process of defining metrics and specifications, persist the results in *verification jobs* and send them to SQuaSH (see `SQR-019 <https://sqr-019.lsst.io/>`_ for a demonstration).

Currently, the following pipelines run in our Jenkins CI system:

- `validate_drp <https://ci.lsst.codes/blue/organizations/jenkins/sqre%2Fvalidate_drp/activity>`_
- `ap_verify <https://ci.lsst.codes/blue/organizations/jenkins/scipipe%2Fap_verify/activity>`_


Storing results in SQuaSH
-------------------------

The SQuaSH REST API is a web app implemented in Flask for managing the SQuaSH metrics dashboard.

When a verification job is sent to the SQuaSH API, the context information like metrics definition and specifications, execution environment metadata, etc are stored in the context database. The SQuaSH context database is a relational database using MySQL (see :ref:`context-database`).

The metric values and associated metadata (the time-series data) are stored primarily in a time-series database using InfluxDB. The resulting *data model* is exposed through the Chronograf UI, and is used for querying the data and visualizing the time series.


Metric values
^^^^^^^^^^^^^

In ``lsst.verify``,  a verification package groups a set of the metric definitions and specifications. See for example `verify_metrics <https://github.com/lsst/verify_metrics/tree/master/metrics>`_.

In InfluxDB, we do not store metric definitions and specifications, only the metric names and values for a given verification package.

Every verification package corresponds to an InfluxDB measurement, and it is a ``string``. The metrics are the fields associated to that measurement. The field key is the full qualified metric name always a ``string``, for example ``validate_drp.AM1``, and the field value is the metric value, usually a ``float``.

Metadata
^^^^^^^^

A verification job includes metadata for the metrics and the execution environment.

The decision of what metadata gets stored as InfluxDB tags and what gets stored as InfluxDB fields follows the general recommendations for the `InfluxDB schema design <https://docs.influxdata.com/influxdb/v1.7/concepts/schema_and_data_layout/>`_ and the queries we need to do also drive that decision.

In InfluxDB, tag values are always ``string``, field values are usually ``float``. See all possible `InfluxDB datatypes <https://docs.influxdata.com/influxdb/v1.7/write_protocols/line_protocol_reference/#data-types>`_.

Examples of job metadata that gets stored as tags are ``pipeline``, ``dataset``, ``filter``, ``ccdnum``, ``visit``, ``patch``, ``tract`` or information that is commonly-queried metadata. An example of job metadata that gets stored as a field is the ``run_id`` because it would increase the `time series cardinality <https://docs.influxdata.com/influxdb/v1.7/concepts/schema_and_data_layout/#don-t-have-too-many-series>`_ otherwise.

The concept of `series <https://docs.influxdata.com/influxdb/v1.7/concepts/glossary/#series>`_ in InfluxDB is also of importance here, every measurement and tag set result in a different series in InfluxDB, and the associated fields make a `point <https://docs.influxdata.com/influxdb/v1.7/concepts/glossary/#point>`_ in every series.

.. note::

  It is not possible to combine tags or fields from different measurements in the same query in InfluxDB, given the above data model this fact should be taken into account when creating new verification packages in ``lsst.verify``.


The following table describes metadata stored as tags. It is not meant to be exhaustive as SQuaSH adds arbitrary metadata present in a verification job as tags by default.

.. csv-table:: The table describe metadata stored as tags.
   :header: Tag key, Example of tag value, Description
   :widths: 20, 20, 50

   env_name, jenkins,  Name of the environment where the metrics were collected
   pipeline, validate_drp, Name of the pipeline that collected the metrics
   instrument, HSC, Name of the instrument that collected the data
   dataset, validation_data_hsc,  Name of dataset that was processed
   filter, HSC-I ,  The corresponding filter for the dataset
   ccdnum,  10,  The ccd ID
   visit, 1543, The visit ID
   patch, 9615, The patch ID
   tract, 1258, The tract ID
   status, 0, Status of the pipeline execution. 0-success or 1-failure

The following table describes metadata stored as fields. SQuaSH explicitly adds these metadata as fields as opposed to tags.

.. csv-table:: The table describes metadata stored as fields.
    :header: Field key, Example of field value, Description
    :widths: 20, 20, 50

    timestamp,1553859000,Timestamp of the pipeline run in Unix time format. It is added as a field to facilitate math operations which are not possible with the original timestamps in InfluxDB.
    run_id, 1612, ID of the pipeline run
    run_url, `1612 <https://ci.lsst.codes/job/sqre/job/validate_drp/1612/>`_, URL of the pipeline run
    squash_id, 3631, ID of the corresponding verification job in SQuaSH
    squash_url, `3631 <https://squash-restful-api.lsst.codes/job/3631>`_, URL of the corresponding verification job in SQuaSH
    code_changes, afw, List of packages that changed w.r.t the previous ``run_id``. It is present in the `jenkins` environment only.
    code_changes_counts, 7, Number of packages that changed w.r.t the previous ``run_id``. It is present in the `jenkins` environment only.


Time-series visualization with Chronograf
-----------------------------------------

The SQuaSH data model is presented to the users through the UI which is based on `Chronograf <https://www.influxdata.com/time-series-platform/chronograf/>`_. From the Chronograf UI, the user can query the metric values, the associated metadata, aggregate results and present them in interactive dashboards.


.. figure:: _static/datamodel.png
       :name: Chronograf UI, graph view.
       :target: _static/datamodel.png

       The figure shows how the SQuaSH data model is presented to the user through the Chronograf UI. This particular query displays a time series for the ``validate_drp.AM1`` metric, selecting the ``validation_data_hsc`` dataset and grouping the results by filter.


.. figure:: _static/datamodel2.png
      :name: Chronograf UI, table view.
      :target: _static/datamodel2.png

      The same query as shown in the previous figure, but now adding the ``run_id`` and visualizing the results in a table view.


Alert rules and notifications with Kapacitor
--------------------------------------------

Chronograf is also the user interface for Kapacitor, a native data processing engine that can process both stream and batch data from InfluxDB. Although, the user can create alerts through the UI the goal is to create alert rules programmatically from the metric specifications stored in SQuaSH.


Using SQuaSH with the LSST Science Platform
-------------------------------------------



Appendix
========

Supporting multiple execution environments
------------------------------------------

To be generally useful for the verification activities, SQuaSH must support multiple execution environments.

Examples of metadata for different execution environments:

   * Jenkins CI
      * Look up key: ID of the CI run
      * Environment metadata: ``ci_id``, ``ci_name``, ``ci_dataset``, ``ci_url``, stack packages.
   * LSST Data Facility (LDF)
      * Look up key: ID of the pipeline run.
      * Environment metadata: run ID, pipeline name, pipeline configuration, butler repository
   * User local environment
      * Look up key: ID of the user run
      * Environment metadata: run ID

The SQuaSH API provides a generic resource to interact with verification jobs, ``/job/<id>``, and specific resources to interact with **runs** on different execution environments. A run may contain results of multiple verification jobs.  For example a ``GET`` request to ``/jenkins/<run_id>`` or to ``/local/<username>/<run_id>`` will retrieve the corresponding jobs.


.. _context-database:

The SQuaSH context database
---------------------------

We adopted a relational database for the SQuaSH context database. The motivation for this choice is mainly for the deploy of the SQuaSH context database to the LSST consolidated database, and the common TAP interface to access the SQuaSH metrics.

In its current deployment, SQuaSH uses a MySQL 5.7 instance in Google Cloud SQL.  MySQL 5.7 offers support to JSON data types which are used to make the database schema more flexible. We store verification job metadata, environment metadata as well as metric definitions and specifications as JSON data types.

Current SQuaSH context database:

   * Entities:
      * ``env``, ``user``, ``job``, ``package``, ``blob``, ``measurement``, ``metric``, ``spec``
   * Relationships:
      * ``1 env : N jobs``
      * ``1 job : N packages``
      * ``1 job : N measurements``
      * ``M measurements : N data blobs``
      * ``1 metric : N specs``
      * ``1 metric : N measurements``


.. figure:: _static/qc-0-db.png
   :name: SQuaSH context database.
   :target: _static/qc-0-db.png

   The figure shows the relational schema for the SQuaSH context database.


In the Google platform deployment, the Cloud SQL manages the backups of the SQuaSH context database.
