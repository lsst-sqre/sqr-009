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

The selected technologies prioritize the use of Python as the 
main development language for rapid prototyping, and the use of the 
selected framework features as much as possible. The main visualization needs,
as summarized at https://dev.lsstcorp.org/trac/wiki/Winter2014/Design/DataAnalysisToolkit
were also taken into consideration.

TODO: Summarize some visualiation requirements here.

The web application is being developed in Django  and we expect less work
on this part of the project as the main structure of the web application 
is done. The QA database is modeled using the object-relational mapper 
(ORM) built in the Django framework.

The Bootstrap framework for web styling is very popular, it supports 
responsive pages on all sorts of devices and can easily be used in combination 
with Django.

Finally, we expect the main development in extending this prototype to 
happen in the bokeh plotting library and datashader to
create interactive visualization, in the QA database model and in the QA 
analysis codes.  

For FITS image visualization we plan to use FFTools JS API to open individual
ccd imagesi linked from the dashboard. 

Still, for sky visualization we plan to integrate Aladin Lite JS plugin. 
In Aladin, the processed dataset images must be pre-rendered in HiPS format 
(http://aladin.u-strasbg.fr/hips/) 
which demands some processing but we benefit from a number of reference survey 
images available as well as the source catalog and polygon overlay features. 

Initially, the QA analysis code will be _afterburner_ scripts that run on
the output of the LSST stack processing. The implementation of the QA workflow 
and parallelization will be discussed in a separate document.


Components
==========

The main components of the SQUASH dashboard prototype are shown in figure 1. 
The figure shows the integration of the QA analysis code with the Django
web application, the Bokeh-server and the QA Database through the ORM layer. 

.. figure:: _static/components.png
   :name: fig-components
   :target: _static/components.png
   :alt: Main components of the SQUASH prototype 

   Main components of SQUASH dashboard prototype.

For development all these components run on a local computer, but if
development requires larger data volumes we can imagine
a situation were the QA database, and perhaps the bokeh server run on a remote 
node.

Implementation Phases
=====================

Phase 1: Initial project structure
    - Create the Django project and initial web application
    - Integration of bokeh server with Django
    - Model dataset, visit and ccd tables in the Django ORM layer
    - Implement template code to compute QA results
    - Implement template code for registration of the QA results in the database
    - Ability to display available datasets
    - Ability to select a dataset and display QA results for each visit in a table
    - Implement a diagnostic plot showing processing status

Phase 2: Adding more interactions to the dashboard
    - Ability to select a visit from a list or from a plot
      and display the focal plane with summary information for each ccd 
      (color coded)
    - Ability to navigate through the list of visits
    - Ability to display QA plots at the visit level
    - Ability to select a ccd and display QA plots at the ccd level
    - Model metrics tables in the Django ORM layer
    - Ability to display metrics at ccd and visit levels

Phase 3: Adding support to multiple runs
    - Model processing tables in the Django ORM layer
    - Ability to display and select available runs for each dataset
    - Ability to access process information

Project structure
=================

.. code-block:: text

    .
    ├── dashboard
    │   ├── admin.py
    │   ├── __init__.py
    │   ├── migrations
    │   │   ├── 0001_initial.py
    │   │   ├── __init__.py
    │   ├── models.py
    │   ├── tests.py
    │   └── views.py
    ├── db.sqlite3
    ├── manage.py
    └── squash
        ├── __init__.py
        ├── settings.py
        ├── urls.py
        └── wsgi.py


References
==========

 - Rapid Prototyping
 - Bokeh webminar
 - Dashboard webminar
 - HiPS: http://aladin.u-strasbg.fr/hips/


APPENDIX A - Making of the squash project
===============================================

In this appendix we document the initial steps used to create
the Django project and the integration with the bokeh-server. 

Python Package Requirements 
---------------------------

We want to use a few more Python packages than the ones mentioned above:

    - Python 3.4.4
    - Django 1.8.4
    - Bootstrap 3.3.6
    - WebTest 2.0.16
    - django-webtest 1.7.7
    - Bokeh 0.11
    - Datashader 0.1

TODO: try to install everything with pip instead of conda, create a virtualenv.

Creating the project
--------------------

.. code-block:: text

    $ django-admin.py startproject squash
    $ cd squash

Running this command creates a new directory called squash, there is a manage.py file which is used to manage a number of aspects of the Django application such as creating the database and running the development web server.  Two other files are squash/settings.py which contains configuration information for the application such as how to connect to the database and squash/urls.py which maps URLs called by the browser to the appropriate Python code.

Since we don't want user authentication in this prototype, we removed the 'django.contrib.auth' from INSTALLED_APPS in squash/settings.py.  

TODO: review this part, other "default" apps could be removed as well

Setting up the database
-----------------------

.. code-block:: text

    $ python manage.py migrate
    $ python manage.py createsuperuser

After running this command, there will be a database file db.sqlite3 in the same directory as manage.py. SQLite works great for development, in production we will probably use MySQL. This command looks at INSTALLED_APPS in squash/settings.py and creates database tables for models defined in those apps models.py files.


Creating the dashboard app
--------------------------

Every Django model must live in an app, so at least one app is needed in a project.

.. code-block:: text

    $ python manage.py startapp dashboard
 

Creating the dashboard models
-----------------------------

Let's create the Datasets, Visit and Ccds tables in the database (as outlined 
in Phase 1) by writing the corresponding classes in dashboard/models.py file. 
Then creates the database tables by running:

.. code-block:: text

    $ python manage.py makemigrations
    Migrations for 'dashboard':
        0001_initial.py:
            - Create model Ccd
            - Create model Dataset
            - Create model Visit
            - Add field visitId to ccd

.. code-block:: text

    $ python manage.py migrate
    Operations to perform:
      Synchronize unmigrated apps: staticfiles, messages
      Apply all migrations: sessions, admin, auth, contenttypes, dashboard
    Synchronizing apps without migrations:
      Creating tables...
        Running deferred SQL...
      Installing custom SQL...
    Running migrations:
      Rendering model states... DONE
      Applying dashboard.0001_initial... OK

Migrations are Django’s way of managing changes to models and the corresponding database. In order to see these
tables from the Django admin interface we need to register them. We can do this by modifying dashboard/admin.py:

.. code-block:: text

    from django.contrib import admin
    from .models import Dataset, Visit, Ccd
    
    admin.site.register(Dataset)
    admin.site.register(Visit)
    admin.site.register(Ccd)

Start up the development server and navigate to the admin site http://localhost:8000/admin/

.. code-block:: text

    $ python manage.py runserver


Integrating Bokeh with Django models
====================================



APPENDIX B - Prototype layout and navigation
============================================


Basic Styling
-------------

The static directory in the top-level directory contains the bootstrap CSS and Javascript
files, it is defined in the squash/settings.py file:

.. code-block:: text

    STATICFILES_DIRS = (
        os.path.join(BASE_DIR, 'static'),
        )

The bootstrap was downloaded from http://getbootstrap.com/getting-started/#download 
and extracted in the static directory, it provides the basic styling for the website.

Prototype layout
----------------

When creating a website it is useful to prototype the 
layout of the pages first, even if the backend is not complete. This section 
explains the mechanism implemented in squash to do that. 

The pages directory contains the prototype pages, it is referenced
using a settings variable in squash/settings.py:

.. code-block:: text

    SITE_PAGES_DIRECTORY=os.path.join(BASE_DIR, 'pages')
    ...

The URL structure implemented in squash/urls.py matches the files in the pages 
directory and loads their contet. With that it's easy to add new prototpype 
pages and have dynamic links to them.

For example, in pages/index.html the code


.. code-block:: text

     href="{% url 'page' 'datasets' %}"

looks for the  pages/datasets.html file. See below example of prototype pages.

.. figure:: _static/home.png
   :name: fig-components
   :target: _static/home.png
   :alt: Prototype layout for SQUASH home
    
   Prototype layout for SQUASH home 

.. figure:: _static/datasets.png
   :name: fig-components
   :target: _static/home.png
   :alt: Datasets page of the SQUASH prototype 
    
   Prototype layour for SQUASH datasets


APPENDIX C - Extending the prototype
====================================

Adding a new plot to the dashboard
----------------------------------

Adding new ccd property at and display 
--------------------------------------

   - Edit the models.py and the new property in the Ccd class
   - Use Django to generate a new migration 
   - Change the QA script to register the new property
   - Add the new property in the views.py
   - Display the new property in a table or plot

Adding a new tab in the Datasets page
-------------------------------------

Adding a new page to the webapp
-------------------------------


