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

     Caption text.

   Run: ``make html`` and ``open _build/html/index.html`` to preview your work.
   See the README at https://github.com/lsst-sqre/lsst-technote-bootstrap or
   this repo's README for more info.

   Feel free to delete this instructional comment.

:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. Add content below. Do not include the document title.

.. note::

   **This technote is not yet published.**

   Report on the SuperTask design as emerging from the 2017 SuperTask Working Group activit.

.. _overview:

Overview
========

.. _task_config_context:

Task/Config Context
===================

.. _functional_design:

Functional Design and Usage Pattern
===================================

- Covers the concept that SuperTasks define their own data-grouping rules
- Specification of inputs and outputs

.. _supertask_interface:

SuperTask class interface
=========================

.. _pipeline_interface:

Pipeline class interface
========================

.. _data_id_mapping:

DataID-mapping model
====================

.. _preflight:

Pre-flight environment
======================

(in particular, the design and behavior that's common across all the implementations)

- “Science DAG” definition
- using the DataID-mapping tool to implement defineQuanta
- logic to produce the “Science DAG” from calls to defineQuanta

.. _quantum_execution:

Quantum-execution environment
=============================

(in particular, the design and behavior that's common across all the implementations)

.. _implementations:

Notes on specific expected implementations
==========================================

(of the Pre-flight and Quantum-execution environments)

- CmdLineActivator
- DRP production
- SUIT / Firefly / Science Platform Portal Aspect use of SuperTask
(open to adding others)

.. _butler_interaction:

Consequential requirements on Butler to support SuperTask
=========================================================

(and description of how Butler is expected to be used in the SuperTask framework)

.. _examples:

Worked examples
===============

- ISR
- Coaddition

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :encoding: latex+latin
..    :style: lsst_aa
