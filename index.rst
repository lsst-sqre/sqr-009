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

The verification of the LSST software stack performance on simulated and precursor data sets is an important activity during the LSST construction. It gives us (SQuaRE) the opportunity to develop Quality Control (QC) infrastructure to support the development of the stack and to preserve the Quality Analysis (QA) code implemented by the DM Science Pipelines group.

The DM Science Data Quality Assurance (SDQA) System Conceptual Design (see `LDM-522 <http://ls.st/LDM-522>`_) describes the QA activities and QC services necessary to implement the capabilities listed in the LSST Data Quality Assurrance plan (see `LDM-63 <http://ls.st/LSE-63>`_). In particular, it describes the different tasks to be performed in each QC tier, from **QC Tier 0** which aims to test and verify the DM sub-system during software development, to **QC Tier 3** which will enable the community to evaluate the data quality of their own analyses.

In the current implementation of SQuaSH we are focused on **QC Tier 0** tasks. For that we compute  Key Performance Metrics (KPMs) on *fixed* data sets through the DM CI system using daily builds of the LSST software stack and send these results to a metrics dashboard that is used to monitor the stability of the code.

The KPMs are computed by afterburner packages such as ``validate_drp`` (see  `DMTN-008 <http://dmtn-008.lsst.io/en/latest/>`_) implemented using the `LSST Verification Framework <https://sqr-019.lsst.io>`_.\


The current implementation of the SQuaSH metrics dashboard can be found at https://squash.lsst.codes/.


We expect to get early feedback from users and iterate with the science pipelines group to improve this system. The main goal is to anticipate the needs for commissioning and and leverage the production SDQA system based on the experience of verifying the LSST software stack during construction.



SQuaSH in the context of the DM software development
====================================================


.. figure:: _static/overview.png
   :name: overview
   :target: _static/overview.png
   :alt: SQuaSH in the context of CI

In the current implementation, SQuaSH supports verification packages that run through the CI system and stores the corresponding metrics, measuremen    ts and associated data (data blobs).



User registration
-----------------


Support for multiple verification packages
------------------------------------------



Sending a verification job to SQuaSH
------------------------------------

First install the `LSST Science Pipelines with lsstsw <https://pipelines.lsst.io/install/lsstsw.html>`_. Specifically, build and setup the verify package:

.. code-block:: bash

   rebuild verify
   # tag this build as current
   eups tags --clone bNNNN current

   # set up the package with EUPS
   setup verify


Assuming you have an output of ``lsst.verify``, e.g. ``Cfht_output_r.json`` you can reproduce the JSON document created by ``dispatch_verify`` in the ``jenkins`` environment using:


.. code-block:: bash

   $ dispatch_verify.py --test --env jenkins --lsstsw $(pwd) Cfht_output_r.json --write test_verify.json




Support for multiple execution environments
-------------------------------------------
In order to be useful for the verification activities SQuaSH must support multiple execution enviroments like the Jenkins CI, the user local environment, the verification cluster environment and potentially other environments. As a consequence, the information displayed in the dashboard will change accordingly to the execution environment.

In order to support multiple execution environments, the environment metadata in a *verification job* must map the corresponding job as suggested below:


   * Jenkins CI
      * Look up key: ID of the CI run
      * Environment metadata: ``ci_name``, ``ci_dataset``, ``ci_label``, ``ci_url``, lsstsw and extra packages
   * User local environment (imply support to multiple users)
      * Look up key: ID of the user run
      * Environment metadata: lsstsw and extra packages
   * Verification Cluster
      * Look up key: ID of the verification cluster run.
      * Environment metadata: lsst stack build (assuming we are using stable versions of the stack only)

.. note::
    For **QC Tier 1** we'll need the ability to save the stack configuration used in each run.


The SQuaSH API provides a generic resource to interact with jobs, ``/jobs/<job_id>`` and specific resources to interact with runs on different execution environments such as Jenkins CI runs that ultimately map to ``jobs``. For example, a request to ``/jenkins/<ci_id>`` or ``/local/<username>/<run_id>`` will look up for the corresponding job to retrieve the associated measurements and metadata.



Commissioning Extensions for SQuaSH
===================================
   * https://confluence.lsstcorp.org/display/LSSTCOM/Commissioning+Extensions+for+SQuaSH



Appendix
========


The QC Tier 0 database
----------------------


For the QC Tier 0 DB, we opted for a relational database because the QC DBs will be deployed to the Oracle *consolidated database* as part of the LSST DAC. SQuaSH currently uses an instance of MySQL 5.7 deployed to Cloud SQL. We choose MySQL over MariaDB because of the support to JSON data types which are used in this implementation to make the database schema more generic. We store Job metadata, environment metadata as well as metric and specification properties as JSON blobs.

Current SQuaSH database schema for QC Tier 0 tasks. This implementation supports multiple verification packages and multiple execution environments.

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
   :name: QC Tier 0 Database
   :target: _static/qc-0-db.png
   :alt: QC Tier 0 Database

Back ups of the SQuaSH QC Tier 0 DB are automated in Cloud SQL.

The SQuaSH RESTful API
----------------------

The SQuaSH RESTful API is a web app implemented in Flask for managing the SQuaSH metrics dashboard.

Current version
^^^^^^^^^^^^^^^

By default, all requests to https://squash-restful-api-demo.lsst.codes/ receive version 1.0 (default) of the RESTful API. The default version of the API may change in the future, thus we encourage you to explicitly request versions via the Accept header.

You can specify a version like this:

.. code-block:: json

    Accept: application/json; version=1.0


Schema
^^^^^^

All API access is over HTTPS, accessed from the https://squash-restful-api-demo.lsst.codes/. All data is sent and received
as JSON.

Authentication
^^^^^^^^^^^^^^

Operations like POST and DELETE (see below) require authentication. To authenticate through the SQuaSH RESTful API you need to provide a valid access token in the authorization header, which can be obtained from the `/auth` endpoint for a registered user:

.. code-block:: python

    import requests

    # assuming a registered user
    user = {'username': user, 'password': passwd}
    r = requests.post("https://squash-restful-api-demo.lsst.codes/auth", json=user)
    access_token = 'JWT ' + r.json()['access_token']

    # assuming you a have a job document you want to post to SQuaSH
    headers = {'Authorization': access_token}
    r = requests.post("https://squash-restful-api-demo.lsst.codes/job", json=job, headers=headers)


Documentation
^^^^^^^^^^^^^

The SQuaSH RESTful API follows the `OpenAPI 2.0 documentation specification <https://github.com/OAI/OpenAPI-Specification/blob/master/versions/2.0.md>`_. The specification is extracted from the docstrings by the `flasgger <https://github.com/rochacbruno/flasgger>`_ utility which is also used to create the `Swagger UI <https://squash-restful-api-demo.lsst.codes/apidocs>`_ for the API.

.. note::
    The Swagger UI is experimental, authentication does not work through this interface yet.

This `notebook <https://github.com/lsst-sqre/squash-rest-api/blob/master/tests/test_api.ipynb>`_ provides an example on how
to interact with the SQuaSH RESTful API from registering a new user in SQuaSH to loading a verification job.

All the available resources and possible operations are listed below:

.. openapi:: _static/apispec_1.json



Deployment
----------

SQuaSH is currently deployed to a commodity cloud (Google Cloud Platform) on Kuberneted and is architected as independent microservices:


.. figure:: _static/squash-deployment.png
   :name: squash-deployment
   :target: _static/squash-deployment.png
   :alt: SQuaSH Kubernetes deployment


The general instructions to deploy squash can be found at `squash-deployment <https://github.com/lsst-sqre/squash-deployment>`_ with links to the individual microservices:

   * `squash-restful-api <https://github.com/lsst-sqre/squash-rest-api>`_: it is used to manage the SQuaSH metrics dashboard. The SQuaSH RESTful API was developed initially using `Django DRF <https://github.com.lsst-sqre/squash-api>`_ and rewrited in Flask. It also includes a Celery app to enable the execution of tasks in background.

   * `squash-bokeh <https://github.com/lsst-sqre/squash-bokeh>`_: serve the squash bokeh apps, we use the `Bokeh plotting library <http://bokeh.pydata.org/en/latest>`_ for rich interactive visualizations.

   * `squash-dash <https://github.com/lsst-sqre/squash-dash>`_: dashboard to embed the bokeh apps. Alternatively we are exploring the possibility to embed the same apps in the Jupyter Lab environment of the LSST Science Platform.


