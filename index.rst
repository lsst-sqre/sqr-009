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

Introduction
============

This document describes the implementation of a prototype dashboard for the
Science Quality Analysis Harness (SQUASH) system.

As stated in http://sqr-008.lsst.io the verification datasets use case 
gives us the oportunity to leverage
QA tests done in the past with pipeQA and more recently with HSC and CFHT QA 
scripts in a comprehensible environment to preserve the codes and practices developed 
by the verification datasets group.

The development will follow the rapid prototype workflow to reach this goal more
efficiently. In this process we build the initial interfaces, discuss its 
layout and content, and as soon as we have a minimal viable product we ship 
the code for use and test purposes. In this process, we will take users 
feedback and iterate back to face usability and performance issues trying 
to engage them in the development. The ultimate goal
is to antecipate SQUASH needs for comissioning and to provide feeback to 
the production system design based on the experience in analyzing precursor 
datasets.

Selecting the right technology stack
====================================

The selected technologies prioritize the use of Python (3.x) as the 
main development language for rapid prototyping, and the use of the 
selected framework features as much as possible. The main visualization needs,
as summarized at https://dev.lsstcorp.org/trac/wiki/Winter2014/Design/DataAnalysisToolkit
were also taken into consideration.

The web application is being developed in Django (1.8.x) and we expect less work
on this part of the project as the main structure of the web application 
is done. The QA database is modeled using the object-relational mapper 
(ORM) built in the Django framework.

The Bootstrap framework for web styling is very popular, it supports 
responsive pages on all sorts of devices and can easily be used in combination 
with Django.

Finally, we expect the main development in extending this prototype to 
happen in the bokeh plotting library (1.1) and datashader (0.1) to
create interactive visualization, in the QA database model and in the QA 
analysis codes.  

For FITS image visualization we plan to use Firefly image viewer. Still, 
for sky visualization we plan to integrate Aladin Lite JS plugin.
In Aladin, the processed dataset images must be pre-rendered in HiPS format, 
which demands some processing but we benefit from a number of reference survey 
images available as well as the source catalog and polygon overlay features. 

Initially, the QA analysis code will be _afterburner_ scripts that run on
the output of the LSST stack processing. The implementation of the QA workflow 
and parallelization will be discussed in a separate document.

Components
==========

The main components of the SQUASH dashboard prototype are shown in figure 1. The figure 
shows the integration of the QA analysis code with the Django
web application, the Bokeh-server and the QA Database through the ORM layer. 

.. figure:: _static/components.png
   :name: fig-components
   :target: _static/components.png
   :alt: Main components of the SQUASH prototype 

   Main components of SQUASH dashboard prototype.

Implementation Phases
=====================

Phase 1: Initial project structure
    - Create the Django project and initial web application
    - Integration of bokeh server with Django
    - Model dataset, visit and ccd tables as in the Django ORM layer
    - Implement template code to compute QA results
    - Implement template code for registration of the QA results in the database
    - Ability to display available datasets
    - Ability to select a dataset and display QA results for each visit
    - Implement a diagnostic plot showing processing status

Phase 2: Adding more interactions to the dashboard
    - Ability to select a visit from a list or from a plot
      and display the focal plane with summary information for each ccd 
      (color coded)
    - Ability to navigate through the list of visits
    - Ability to display QA plots at the visit level
    - Ability to select a ccd and display QA plots at the ccd level
    - Model metrics tables of the QA database
    - Ability to display metrics at ccd and visit levels

Phase 3: Adding support to multiple runs
    - Model processing tables in the Django ORM layer
    - Ability to display and select available runs for each dataset
    - Ability to access process information

Project structure
=================

.. code-block:: text

    .
    ├── conda-requirements.txt
    ├── pages
    ├── squash.py
    ├── viz
    └── webapp
     ├── __init__.py
     ├── static
     │   ├── css
     │   ├── fonts
     │   └── js
     ├── templates
     ├── urls.py
     ├── views.py
     └── models.py


Extending the prototype
=======================

Adding a new plot to the dashboard
----------------------------------

Adding a new property to the ccd table and display 
--------------------------------------------------

   - Edit the models.py and the new property in the corresponding classes
   - Use Django to generate a new migration 
   - Change the QA script to register the new property
   - Add the new property in the views.py
   - Display the new property in a table or plot

Adding a new page to the webapp
-------------------------------
