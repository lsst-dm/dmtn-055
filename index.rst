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

The design of the SuperTask Library is largely derived from the following two principles:

 - Defining units of work that can be performed independently should be a responsibility of the same class (a concrete ``SuperTask``, in this case) that does that work.  Putting this responsibility on the control software or the human user instead would result in a rigid system that is capable of running only a few predefined sequences of ``SuperTasks`` without requiring significant changes.

 - By requesting a list of these units of work from each ``SuperTask`` in an ordered list, the control software can discover all dependencies and construct a satisfactory execution plan, in advance, for the full sequence of ``SuperTasks``.  This does not allow the definition of a particular ``SuperTask's`` units of work to depend on the details of the outputs of an earlier ``SuperTask`` in the sequence (as opposed to depending on just the presence or absenct of outputs).

We consider this limitation acceptable for two reasons.  First, we expect cases where the details of the outputs affect the dependencies to be rare, and hence it is an acceptable fallback to simply split the list of ``SuperTasks`` into subsets without these dependencies and run the subsets in sequence manually, because the number of such subsets will be small.  More importantly, we believe we can strictly but only slightly overestimate the dependencies between units of work in advance, in essentially all of these cases, and hence the only errors in the execution plan will be a small number of do-nothing jobs and/or unnecessary inputs staged to the local compute environment.  These can easily be handled by any practical workflow system.

For the remainder of this document, we will refer to an independent unit of work performed by a ``SuperTask`` (and the list of input and output datasets involved) as a *Quantum*.  An ordered list of ``SuperTasks`` (which includes their configuration) is what we call a *Pipeline*.  The control software has many components with different responsibilities, which we will introduce in the remainder of this section.

The typical usage pattern for the SuperTask Library is as follows.

#.  A developer defines a ``Pipeline`` from a sequence of ``SuperTasks``, including their configuration, either programmatically or by editing a TBD text-based, human-readable file format.  Other developers may then modify the ``Pipeline`` to modify configuration or insert or delete ``SuperTasks``, again via either approach.

#.  An operator passes the ``Pipeline``, an input data repository to a ``PreflightFramework``, and a Data ID Expression (see :ref:`data_id_mapping`).  Different ``PreflightFrameworks`` will be implemented for different contexts.  Some ``PreflightFramework``s may provide an interface for making a final round of modifications to the ``Pipeline`` at this stage, but these modifications are not qualitatively different from those in the previous step.

#.  The ``PreflightFramework`` passes the ``Pipeline``, the input data repository, and the Data ID Expression to a ``GraphBuilder`` (see :ref:`preflight`), which

    - inspects the ``Pipeline`` to construct a list of all dataset types consumed and/or produced by the ``Pipeline``;
    - queries the data repository to obtain a ``RepoGraph`` that contains all datasets of these types that match the given Data ID Expression (see :ref:`data_id_mapping`);
    - calls the ``defineQuanta`` method of each ``SuperTask`` in the ``Pipeline`` in sequence, accumulating a list of all quanta to be executed;
    - constructs the Science DAG (see :ref:`preflight`), a bipartate directed acyclic graph with ``Quantum`` vertices linked by the dataset vertices they produce and consume.

#.  The Science DAG is passed to an ``ExecutionFramework``, along with additional configuration for how the processing is to be performed (changes in this configuration must not change the outputs of the ``Pipeline`` except to allow intermediate datasets to be elided).  The ``ExecutionFramework`` may be the same class as the ``PreflightFramework`` (as in ``CmdLineTask``, which performs both roles), which makes this step a no-op.  It may also be a completely different class that may be run in an entirely different compute environment (via a serialized Science DAG).

#.  The ``ExecutionFramework`` creates one or more output data repositories and records in them any repository-wide provenance (such as the ``Pipeline`` configuration or software versions).

#.  The ``ExecutionFramework`` walks the Science DAG according to the partial ordering it defines, and calls ``runQuantum`` on the appropriate concrete ``SuperTask`` for each ``Quantum`` vertex.  Depending on the activator, the ``SuperTasks`` may be run directly in the same compute environment, or submitted to a workflow system for execution elsewhere (probably by translating the generic Science DAG to a format specific to a particular workflow system).  In some environments a temporary local data repository containing only the datasets consumed by a particular set of quanta may be created in scratch space to support execution in a context in which the original data repositories are not accessible, with output datasets similarly staged back to the true output data repositories.

.. note::

    The above procedure does not provide a mechanism for adding camera-specific overrides to the configuration.  I think this has to be part of the ``Pipeline`` interface that's done in the first step, not something done later by ``PreflightFrameworks``.  That's especially true if we want to permit ``Pipelines`` that aggregate data from multiple cameras; in that case I think we'd need the `Pipeline` itself to hold the overrides for different cameras in addition to the defaults to avoid spurious provenance issues from having different configurations of the same ``Pipeline`` in a repo.  Given that different cameras might even change the ``SuperTasks`` we want in a ``Pipeline``, we may need to make it possible to parameterize all of a ``Pipeline's`` definition on different Units of data (not just cameras, but filters).  I'm sure that's doable, but it's a lot more complexity than we were imagining when we punted on the details of the ``Pipeline`` API.


.. _supertask_interface:

SuperTask Class Interface
=========================

The declaration for the ``SuperTask`` abstract base class is sufficiently simple that we can simply reproduce it here:

.. code-block:: py

    class SuperTask(Task):

        def __init__(self, butler=None, **kwargs):
            ...

        def run(self, *args, **kwargs):
            raise NotImplementedError()

        def defineQuanta(self, repoGraph):
            raise NotImplementedError()

        def runQuantum(self, quantum, butler):
            raise NotImplementedError()

        def getDatasetClasses(self):
            ...

        def getDatasetSchemas(self):
            ...

.. note::
    This differs from the code in ``pipe_supertask`` a bit (other than just being a summary with no docstrings or implementation):
     - I've rewritten ``__init__``'s signature to use ``**kwds`` to allow it to forward all arguments to the ``Task`` constructor.
     - I've removed the ``butler`` argument from ``defineQuanta``; I don't think it's necessary.
     - I've removed ``write_config`` and ``_get(_resource)_config_name``; I think writing is the responsibility of the ``PreflightFramework``, and I think the config name should always be set from ``_DefaultName`` (which is part of ``Task``, not just ``SuperTask``).
     - Removed ``write_schema`` in favor of ``getDatasetSchemas``.  Again, I think writing should be the responsibility of the ``PreflightFramework``. so we just need a way for it to get the schema(s) from the ``SuperTask``.

Construction
    All concrete ``SuperTasks`` must have the ``__init__`` signature shown here, in which ``**kwargs`` contains only arguments to be forwarded to ``Task.__init__`` (additional keyword-only arguments are also allowed, as long as they have default values).  The abstract base class does not use the ``butler`` argument, allowing it to be ``None``, and while concrete ``SuperTasks`` may or may not use it, they must accept it even if it is unused.  This allows the schemas associated with input dataset types and the configuration of preceeding ``SuperTasks`` to be loaded and used to complete construction of the ``SuperTask``; a ``SuperTask`` should not assume any other datasets are available through the given ``Butler``.  ``SuperTasks`` that do use the ``butler`` argument should also provide an alternate way to provide the schemas and configuration (i.e. additional defaulted keyword arguments) to allow them to be constructed without a ``Butler`` when used as a regular ``Task``.  This also implies that when a ``Pipeline`` constructs a sequence of ``SuperTasks``, it must ensure the schemas and configuration are recorded at each step, not just at the end.

``run(*args, **kwargs)``
    This is the standard entry point for all ``Tasks``, with the signature completely different for each concrete ``Task``.  This should perform the bulk of the ``SuperTask's `` algorithmic work, operating on in-memory objects for both arguments and return values, and should not utilize a ``Butler`` or perform any I/O.  In rare cases, a ``SuperTask`` for which I/O is an integral component of the algorithm may lack a ``run`` method, or may have multiple methods to serve the same purpose.  As with other ``Tasks``, the return value should be a ``pipe.base.Struct`` combining named result objects.

``defineQuanta(repoGraph)``
    Called during :ref:`pre-flight <preflight>`, in this method a concrete ``SuperTask`` subdivides work into independently-executable units (quanta) and relates the input datasets of these to their output datasets.
    The only argument is a :ref:`RepoGraph <data_id_mapping>`` instance, a graph object describing the current state of the relevant subset of the input data repository.  On return, the ``RepoGraph`` should be modified to additionally contain datasets that will be produced by the ``SuperTask``, reflecting the fact that they will be present in the data repository by the time subsequent ``SuperTask``s in the same ``Pipeline`` are executed.  The return value should be a list of :py:class:`Quantum` instances.

``runQuantum(quantum, butler)``
    This method actually runs the ``SuperTask`` on the given :ref:```Quantum`` <quantum_interface>`, using a ``Butler`` for input and output.  For most concrete ``SuperTasks``, this should simply use ``Butler.get`` to retrieve inputs, call ``run``, and then use ``Butler.put`` to write outputs.

.. _quantum_interface

Quantum Class Interface
=======================

``Quantum`` is a simple struct-like class that simply aggregates the input and output datasets for a unit of work that can be performed independently by a ``SuperTask``:

.. py:class:: Qunatum

.. code-block:: py

    class Quantum(object):

        def __init__(self, inputs, outputs):
            self.inputs = inputs
            self.outputs = outputs

The ``inputs`` and ``outputs`` attributes are dictionaries with ``Dataset`` class types as keys and a ``set`` of ``Dataset`` instances (of that type) as values.


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

- CmdLineFramework
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
