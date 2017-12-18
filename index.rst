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

In this document we present the design of the Science Quality Analysis Harness (SQuaSH) metrics dashboard.

The verification of the LSST software stack performance on simulated data and on precursor data sets is an important activity during the LSST construction and gives us (SQuaRE) the opportunity to develop Quality Control (QC) infrastructure to support the development of the stack and to preserve the QA analysis code developed by the DM science pipelines group.

For **QA-0** we compute the Key Performance Metrics (KPMs) on *fixed* data sets through the CI system using the daily builds of the LSST software stack and send these results to a metrics dashboard that is used to monitor the stability of the code.

The KPMs are computed by afterburner packages such as `validate_drp` (see  `DMTN-008 <http://dmtn-008.lsst.io/en/latest/>`_) implemented using the `LSST Verification Framework <https://sqr-019.lsst.io>`_.\


The current implementation of the SQuaSH metrics dashboard can be found at https://squash.lsst.codes/.


We expect to get early feedback from users and iterate with the science pipelines group to improve this system. The main goal is to anticipate the needs for commissioning and and leverage the production SDQA system based on the experience of verifying the LSST software stack during construction.



Architecture
============

SQuaSH is currently deployed to a commodity cloud (Google Cloud Platform) and developed as independent microservices as
shown in figure 1.


.. figure:: _static/squash-deployment.png
   :name: squash-deployment
   :target: _static/squash-deployment.png
   :alt: SQuaSH Kubernetes deployment


The general instructions to deploy squash are found at `squash-deployment <https://github.com/lsst-sqre/squash-deployment>`_ with links to the individual microservices:

   * `squash-db <https://github.com/lsst-sqre/squash-db>`_: currently uses MariaDB 10.3+ but soon will migrate to MySQL 5.7 because of the MySQL support to JSON data types. We opted for a relational database because the QC database will be deployed to the Oracle *consolidated database* that will be part of the DAC.
   * `squash-api <https://github.com/lsst-sqre/squash-api>`_: was developed initially using `Django DRF <http://www.django-rest-framework.org/>`_ but will migrate to Flask soon. It is used to manage the SQuaSH metrics dashboard.
   * `squash-bokeh <https://github.com/lsst-sqre/squash-bokeh>`_: serve the squash bokeh apps, we use the `Bokeh plotting library <http://bokeh.pydata.org/en/latest>`_ for rich interactive visualizations.
   * `squash-dash <https://github.com/lsst-sqre/squash-dash>`_: dashboard to embed the bokeh apps. Alternatively we are exploring the possibility to embed the same apps in the Jupyter Lab environment of the LSST Science Platform.


Implementation Road Map
=======================

Support the verification packages:

   * validate_drp
   * jointcal
   * ap_verify


Drill down capability
=====================


Job execution capability
========================

Commissioning Extensions for SQuaSH
===================================

   * https://confluence.lsstcorp.org/display/LSSTCOM/Commissioning+Extensions+for+SQuaSH


SQuaSH in the context of the LSST Science Platform
==================================================

Appendix
========

The QC-0 database
---------------

The SQuaSH RESTful API
----------------------

Data visualization with bokeh
-----------------------------

Data access investigations
--------------------------

