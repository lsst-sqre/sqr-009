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

We present the current design for the Science Quality Analysis Harness (SQuaSH) metrics dashboard being developed by LSST/DM SQuaRE as part of the verification effort.

The verification of the LSST software stack performance on simulated and precursor data sets is an important activity during the project construction phase. It gives LSST/DM the opportunity to develop Quality Control (QC) infrastructure and Quality Analysis (QA) code to support the development of the stack and evaluate its results.

The DM Science Data Quality Assurance (SDQA) System Conceptual Design [`LDM-522 <http://ls.st/LDM-522>`_] describes the  QC services and QA activities necessary to implement the capabilities listed in the LSST Data Quality Assurrance plan [`LDM-63 <http://ls.st/LSE-63>`_]. In particular, it describes the different tasks to be performed in each QC tier, starting from the **QC Tier 0** which aims to test and verify the DM sub-system during software development, to the **QC Tier 3** which will enable the science collaboration to evaluate the data quality for their own science analyses.

SQuaSH is a QC service, in its current implementation it is focused on **QC Tier 0** tasks. For that we compute  Key Performance Metrics (KPMs) on *fixed* data sets to monitor the stability of the LSST software stack through the Jenkins Continuous Integration (CI) system from daily builds. As part of the CI pipeline, the KPMs are computed by afterburner packages such as ``validate_drp``  [`DMTN-008 <http://dmtn-008.lsst.io/en/latest/>`_] and the results are sent to the SQuaSH metrics dashboard for monitoring.


In addition to that, we propose a workflow for developers to enable them to instrument their specific Science Pipeline Tasks, perform the verification measurements on development branches through the CI system or even on their local execution environment, and send the results to SQuaSH. We also discuss the SQuaSH integration with the LSST Science Platform (LSP) and how users of the notebook aspect of the LSP can benefit from pre-built SQuaSH apps embeded in the Jupyter notebooks. Finally, in the Appendix we provide more information about the system architecture and its deployment on the Google Kubernetes Engine (GKE).

The current production version of the SQuaSH metrics dashboard is available at https://squash.lsst.codes/ and the current development version at https://squash-demo.lsst.codes/.

We expect to get feedback from users and iterate with the science pipelines group to incorporate their metrics and visualizations in SQuaSH. The main goal is to anticipate the needs for commissioning and to leverage the production SDQA system based on this experience.


Design guidelines
=================



Some of the design guidelines and features of SQuaSH are summarized below:

 - implement the concepts developed in the `LSST Verification Framework <https://sqr-019.lsst.io>`_
 - implement a QC database to support the tasks performed in the QC Tier 0, 1, 2 and 3 to preserve and visualize their results
 - support multiple test datasets and multiple verification packages
 - support automated verification runs, initially through the Jenkins CI environment and other execution environments
 - enable interactive visualization for the metric measurements
 - correlate deviations of the metric measurements with code changes
 - provide notifications/alerts upon metric measurement deviations
 - provide the "drill down" capability, i.e, in addition to the visualization of the scalar metrics enable further interactive visualization from the same data used to perform the metric measurements
 - extensible: make it easy for users to add new visualizations (a.k.a SQuaSH apps)
 - embeddable: reuse SQuaSH apps for analysis in the LSP notebook aspect
 - replaceable: easy to replace SQuaSH components keep up with standard technologies
 - deployable on Kubernetes

Other desired capabilities:

 - support verification runs from the developer "local" environment
 - support multi users to keep results from user "local" verification runs, use GitHub OAuth for the SQuaSH API.
 - record measurements of a given metric per Patch, Visit, CCD specially for QC-Tier 2 and 3 where we process larger data sets. If we aggregate metric measurements at the CCD level to the Visit level we can also implement some kind of drill down visualization.


Commissioning Extensions for SQuaSH
-----------------------------------
   * https://confluence.lsstcorp.org/display/LSSTCOM/Commissioning+Extensions+for+SQuaSH



SQuaSH in the context of the DM software development
====================================================


The LSST Verification framework
-------------------------------

The ``lsst.verify`` is the framework for making verification measurements in the LSST Science Pipelines. ``lsst.verify`` aims to generalize the process of defining metrics, measuring those metrics, and tracking those measurements in SQuaSH. The design is described in `SQR-017 <https://sqr-019.lsst.io/>`_, and `SQR-019 <https://sqr-019.lsst.io/>`_ demonstrates its use.

``lsst.verify`` is currently being used by specialized afterburner packages such as ``validate_drp`` [`DMTN-008 <http://dmtn-008.lsst.io/en/latest/>`_] but it can also be used to track performance metrics of specific pipeline Tasks such as ``jointcal``.

See also `DMTN-057 <https://dmtn-057.lsst.io/>`_ for a discussion on how to instrument the Science Pipelines tasks with verification metrics based on the ``ap_verify`` experience.


The SQuaSH API implements the concepts developed in ``lsst.verify``. A verification ``Job`` is the result of a verification run and contains at least one measurement of a particular metric. A ``Job`` also contains metadata, with information about the execution environment, and the actual data used to make the measurements. The data used to make the measurements is called `data blob` which includes tabular data and additional metadata. Blobs are used for the drill down visualization in SQuaSH.

A ``Job`` object produced by ``lsst.verify`` can be persisted as a JSON file and sent to SQuaSH using the ``dispatch_verify`` utility.


Next we discuss use cases for DM developers involving SQuaSH.


Use case 1: Afterburner packages in CI
--------------------------------------

.. figure:: _static/overview.png
   :name: overview
   :target: _static/overview.png
   :alt: SQuaSH in the context of CI

SQuaSH supports verification packages that run as afterburners in Jenkins CI. In the current implementation the Jenkins ``release/nightly-release`` pipeline builds the stack and triggers ``validate_drp`` which measures metrics from the ``ProcessCcdTask`` outputs.

By measuring metrics on `fixed` data sets in CI, we monitor the stability of the LSST software stack and can correlate deviations of the metric measurements with code changes.

The results are sent to the `SQuaSH metrics dashboard <https://squash.lsst.codes/>`_.


Use case 2: Instrumenting science pipeline Tasks
------------------------------------------------

Developers can implement verification metrics on their own pipeline Tasks.

The ``jointcal`` is an example of science pipeline Task that uses ``lsst.verify`` for defining its performance metrics and making verification measurements.

A Task should produce one verification ``Job`` containing the set of measurements performed by the Task. Usually it would be part of a larger pipeline involving several tasks and the verification measurements would be sent to SQuaSH after the execution of each Task.

In order to illustrate this use case we run ``jointcal`` on HSC test data and use ``dispatch_verify`` to sent the results to SQuaSH.


.. code-block:: bash

    $ setup obs_subaru
    $ setup jointcal
    $ setup testdata_jointcal
    $ export ASTROMETRY_NET_DATA_DIR=${TESTDATA_JOINTCAL_DIR}/hsc_and_index
    $ jointcal.py ${TESTDATA_JOINTCAL_DIR}/hsc/ --output output --id visit=0903334^0903336^0903338^0903342^0903344^0903346 --clobber-versions --clobber-config


Which will produce among other things the ``verify_output.json`` file with the verification measurements.

Here we set explicitly some variables that would be required in the Jenkins CI environment, or the equivalent in other execution environments:

.. code-block:: bash

  $ export BUILD_ID=1  # ID in the CI system
  $ export PRODUCT=jointcal # Name of the package
  $ export dataset=hsc # Name of the test dataset used
  $ dispatch_verify.py --url https://squash-restful-api-demo.lsst.codes --user <squash user> --password <squash passwd> --env jenkins --lsstsw <lsstsw directory path> verify_output.json



Use case 3: Supporting development branches
------------------------------------------

Given that specific Tasks can make verification measurements on test data sets (see e.g. the ``jointcal`` tests) and send results to SQuaSH, it might be interesting for the developer to do so before merging the development branch to master. That would enable developers to compare their performance metrics with previous results on master and to avoid regressions in the first place. The results would be sent to SQuaSH when the Jenkins ``stack-os-matrix`` job is triggered by the developer. We can implement on Jenkins a similar mechanism we have to run the stack demo pipeline something like `Send verification measurements to SQuaSH`. SQuaSH can keep track of the branch being tested and the dashboard should make it easy to identify results from development branches and compare them with results for the same verification metrics from master.

  .. note::
    If the execution time of the ``stack-os-matrix`` job becomes impeditive due to new tests executing the verification measurements, we could think about doing the verification measurements on specialized verfication packages that execute the science pipeline Tasks of interest.


Use case 4: Supporting the developer local environment
------------------------------------------------------

Another use case is to support the developer local execution environment. An implication for that is to support multiple users in SQuaSH. Then the ``user_id`` can be associated with the corresponding ``job_id`` and the results can be displayed by user in the SQuaSH dashboard.

   .. note::
    Multi user support is not fully implemented yet. While user registration and token authentication is already present in the SQuaSH API we need a third-party OAuth provider for authentication purposes.

In this scenario, ``dispatch_verify`` would accept the user ``local`` environment with the appropriate metadata such as an incremental user run ID, the data set used and the directory path for the user local ``lsstsw`` installation in order to register the package metadata associated to its run.


Supporting multiple execution environments
==========================================

In order to be useful for the verification activities SQuaSH must support multiple execution enviroments like the Jenkins CI, the user "local" environment, the verification cluster environment and potentially others.

In the verification framework, a ``Job`` packages several measurements, metadata and data blobs. The metadata contains information about the execution environment.

Examples of verification ``Job`` metadata for different execution environments:

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
    For **QC Tier 1** the metadata may include things like the stack configuration used in each run.


The SQuaSH RESTful API provides a generic resource to interact with jobs and specific resources to interact with runs on different execution environments that ultimately map to a job. In this way a request to ``/jenkins/<ci_id>`` or ``/local/<username>/<run_id>`` will look up for the corresponding ``/jobs/<job_id>`` to retrieve the associated measurements, metadata and data blobs.



SQuaSH in the context of the LSST Science Platform
==================================================

.. figure:: _static/squash_lsp.png
   :name: overview
   :target: _static/overview.png
   :alt: SQuaSH in the context of the LSP

Using the LSP environment for QC-Tier 3 analysis. Using SQuaSH to submit verification runs in the verification cluster. Embedding SQuaSH apps in the JupyterLab environment.



Appendix
========


Deployment
----------

SQuaSH is currently deployed to a commodity cloud, the Google Cloud Platform, on the Google Kubernetes Engine (GKE), and is architected as independent microservices. The figure below shows the various "layers" of the Kubernetes deployment, the *service* which provides an external IP to the microservice, the *pod* which groups containers running on the same GKE node. Other Kubernetes objects like *secrets* and customized configurations stored as *configmaps* are also indicated in the figure. The microservices ``squash-restful-api``, ``squash-bokeh`` and ``squash-dash`` are connected through HTTPS and TLS termination is implemented trough the ``nginx`` container which works as a reverse proxy to secure the traffic outside the pods.

.. figure:: _static/squash-deployment.png
   :name: squash-deployment
   :target: _static/squash-deployment.png
   :alt: SQuaSH Kubernetes deployment


The general instructions to deploy squash can be found at `squash-deployment <https://github.com/lsst-sqre/squash-deployment>`_ with links to the individual microservices:

   * `squash-restful-api <https://github.com/lsst-sqre/squash-rest-api>`_: it is used to manage the SQuaSH metrics dashboard. The SQuaSH RESTful API was developed initially using `Django DRF <https://github.com.lsst-sqre/squash-api>`_ and then reimplemented in Flask with several extensions. It also uses Celery to enable the execution of tasks in background. This can be extended later to

   * `squash-bokeh <https://github.com/lsst-sqre/squash-bokeh>`_: it serves the squash bokeh apps, we use the `Bokeh plotting library <http://bokeh.pydata.org/en/latest>`_ for rich interactive visualizations.

   * `squash-dash <https://github.com/lsst-sqre/squash-dash>`_: dashboard to embed the bokeh apps. Alternatively we are exploring the possibility to embed the same apps in the Jupyter Lab environment of the LSST Science Platform.



The QC Tier 0 database
----------------------


For the QC Tier 0 DB, we opted for a relational database. The motivation behind this choice is that we plan to deploy QC DBs to the Oracle *consolidated database* as part of the LSP. SQuaSH currently uses an instance of MySQL 5.7 deployed to Cloud SQL. We chose MySQL over MariaDB, used in Qserv, because of the support to JSON data types which are used in this implementation to make the database schema more generic. We store Job metadata, environment metadata as well as metric definitions and specifications as JSON blobs.

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



