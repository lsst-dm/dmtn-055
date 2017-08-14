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

 - Defining units of work that can be performed independently should be a responsibility of the same class (a concrete :py:class:`SuperTask`, in this case) that does that work.  Putting this responsibility on the control software or the human user instead would result in a rigid system that is capable of running only a few predefined sequences of :py:class:`SuperTasks <SuperTask>` without requiring significant changes.

 - By requesting a list of these units of work from each :py:class:`SuperTask` in an ordered list, the control software can discover all dependencies and construct a satisfactory execution plan, in advance, for the full sequence of :py:class:`SuperTasks <SuperTask>`.  This does not allow the definition of a particular :py:class:`SuperTask's <SuperTask>` units of work to depend on the details of the outputs of an earlier :py:class:`SuperTask` in the sequence (as opposed to depending on just the presence or absenct of outputs).

We consider this limitation acceptable for two reasons.  First, we expect cases where the details of the outputs affect the dependencies to be rare, and hence it is an acceptable fallback to simply split the list of :py:class:`SuperTasks <SuperTask>` into subsets without these dependencies and run the subsets in sequence manually, because the number of such subsets will be small.  More importantly, we believe we can strictly but only slightly overestimate the dependencies between units of work in advance, in essentially all of these cases, and hence the only errors in the execution plan will be a small number of do-nothing jobs and/or unnecessary inputs staged to the local compute environment.  These can easily be handled by any practical workflow system.

For the remainder of this document, we will refer to an independent unit of work performed by a :py:class:`SuperTask` (and the list of input and output datasets involved) as a *Quantum*.  An ordered list of :py:class:`SuperTasks <SuperTask>` (which includes their configuration) is what we call a *Pipeline*.  The control software has many components with different responsibilities, which we will introduce in the remainder of this section.

The typical usage pattern for the SuperTask Library is as follows.

#.  A developer defines a :py:class:`Pipeline` from a sequence of :py:class:`SuperTasks <SuperTask>`, including their configuration, either programmatically or by editing a TBD text-based, human-readable file format.  Other developers may then modify the :py:class:`Pipeline` to modify configuration or insert or delete :py:class:`SuperTasks <SuperTask>`, again via either approach.

#.  An operator passes the :py:class:`Pipeline`, an input data repository to a ``PreflightFramework``, and a Data ID Expression (see :ref:`data_id_mapping`).  Different ``PreflightFrameworks`` will be implemented for different contexts.  Some ``PreflightFrameworks`` may provide an interface for making a final round of modifications to the :py:class:`Pipeline` at this stage, but these modifications are not qualitatively different from those in the previous step.

#.  The ``PreflightFramework`` passes the :py:class:`Pipeline`, the input data repository, and the Data ID Expression to a ``GraphBuilder`` (see :ref:`preflight`), which

    - inspects the :py:class:`Pipeline` to construct a list of all dataset types consumed and/or produced by the :py:class:`Pipeline`;
    - queries the data repository to obtain a ``RepoGraph`` that contains all datasets of these types that match the given Data ID Expression (see :ref:`data_id_mapping`);
    - calls the ``defineQuanta`` method of each :py:class:`SuperTask` in the :py:class:`Pipeline` in sequence, accumulating a list of all quanta to be executed;
    - constructs the Science DAG (see :ref:`preflight`), a bipartate directed acyclic graph with quantum vertices linked by the dataset vertices they produce and consume.

#.  The Science DAG is passed to an ``ExecutionFramework``, along with additional configuration for how the processing is to be performed (changes in this configuration must not change the outputs of the :py:class:`Pipeline` except to allow intermediate datasets to be elided).  The ``ExecutionFramework`` may be the same class as the ``PreflightFramework`` (as in ``CmdLineTask``, which performs both roles), which makes this step a no-op.  It may also be a completely different class that may be run in an entirely different compute environment (via a serialized Science DAG).

#.  The ``ExecutionFramework`` creates one or more output data repositories and records in them any repository-wide provenance (such as the :py:class:`Pipeline` configuration or software versions).

#.  The ``ExecutionFramework`` walks the Science DAG according to the partial ordering it defines, and calls ``runQuantum`` on the appropriate concrete :py:class:`SuperTask` for each quantum vertex.  Depending on the activator, the :py:class:`SuperTasks <SuperTask>` may be run directly in the same compute environment, or submitted to a workflow system for execution elsewhere (probably by translating the generic Science DAG to a format specific to a particular workflow system).  In some environments a temporary local data repository containing only the datasets consumed by a particular set of quanta may be created in scratch space to support execution in a context in which the original data repositories are not accessible, with output datasets similarly staged back to the true output data repositories.

.. note::

    The above procedure does not provide a mechanism for adding camera-specific overrides to the configuration.  I think this has to be part of the :py:class:`Pipeline` interface that's done in the first step, not something done later by ``PreflightFrameworks``.  That's especially true if we want to permit ``Pipelines`` that aggregate data from multiple cameras; in that case I think we'd need the `Pipeline` itself to hold the overrides for different cameras in addition to the defaults to avoid spurious provenance issues from having different configurations of the same :py:class:`Pipeline` in a repo.  Given that different cameras might even change the :py:class:`SuperTasks <SuperTask>` we want in a :py:class:`Pipeline`, we may need to make it possible to parameterize all of a :py:class:`Pipeline's <Pipeline>` definition on different Units of data (not just cameras, but filters).  I'm sure that's doable, but it is not currently supported by the :py:class:`Pipeline` API in this document.


.. _supertask_interface:

SuperTask Class Interface
=========================

.. py:class:: SuperTask(Task)

    .. py:method:: __init__(self, butler=None, **kwargs)

        All concrete :py:class:`SuperTasks <SuperTask>` must have the ``__init__`` signature shown here, in which ``**kwargs`` contains only arguments to be forwarded to ``Task.__init__`` (additional keyword-only arguments are also allowed, as long as they have default values).  The abstract base class does not use the ``butler`` argument, allowing it to be ``None``, and while concrete :py:class:`SuperTasks <SuperTask>` may or may not use it, they must accept it even if it is unused.  This allows the schemas associated with input dataset types and the configuration of preceeding :py:class:`SuperTasks <SuperTask>` to be loaded and used to complete construction of the :py:class:`SuperTask`; a :py:class:`SuperTask` should not assume any other datasets are available through the given ``Butler``.  :py:class:`SuperTasks <SuperTask>` that do use the ``butler`` argument should also provide an alternate way to provide the schemas and configuration (i.e. additional defaulted keyword arguments) to allow them to be constructed without a ``Butler`` when used as a regular ``Task``.  This also implies that when a :py:class:`Pipeline` constructs a sequence of :py:class:`SuperTasks <SuperTask>`, it must ensure the schemas and configuration are recorded at each step, not just at the end.

    .. py:method:: run(self, *args, **kwargs)

        This is the standard entry point for all ``Tasks``, with the signature completely different for each concrete ``Task``.  This should perform the bulk of the :py:class:`SuperTask's <SuperTask>` algorithmic work, operating on in-memory objects for both arguments and return values, and should not utilize a ``Butler`` or perform any I/O.  In rare cases, a :py:class:`SuperTask` for which I/O is an integral component of the algorithm may lack a ``run`` method, or may have multiple methods to serve the same purpose.  As with other ``Tasks``, the return value should be a ``pipe.base.Struct`` combining named result objects.

    .. py:method:: defineQuanta(self, repoGraph)

        Called during :ref:`pre-flight <preflight>`, in this method a concrete :py:class:`SuperTask` subdivides work into independently-executable units (quanta) and relates the input datasets of these to their output datasets.
        The only argument is a :ref:`RepoGraph <data_id_mapping>` instance, a graph object describing the current state of the relevant subset of the input data repository.  On return, the ``RepoGraph`` should be modified to additionally contain datasets that will be produced by the :py:class:`SuperTask`, reflecting the fact that they will be present in the data repository by the time subsequent :py:class:`SuperTask's <SuperTask>` in the same :py:class:`Pipeline` are executed.  The return value should be a list of :py:class:`Quantum` instances.

    .. py:method:: runQuantum(self, quantum, butler)

        This method actually runs the :py:class:`SuperTask` on the given :py:class:`Quantum`, using a ``Butler`` for input and output.  For most concrete :py:class:`SuperTasks <SuperTask>`, this should simply use ``Butler.get`` to retrieve inputs, call :py:meth:`run`, and then use ``Butler.put`` to write outputs.

    .. py::method:: getDatasetClasses(self)

        Called during :ref:`pre-flight <preflight>` (before :py:meth:`defineQuanta`), this method returns the sets of input and output :py:class:`Datasets <Dataset>` classes used by this :py:class:`SuperTask`.  As long as :py:class:`DatasetField <supertask_interface_configuration>` is used to control the :py:class:`Dataset` classes utilized by the :py:class:`SuperTask's <SuperTask>`, the default implementation provided by the :py:class:`SuperTask` base class itself should be sufficient.

    .. py::method:: getDatasetSchemas(self)

        This method returns a dict containing the schemas that correspond to any table-like datasets output by the :py:class:`SuperTask`.  Dictionary keys are :py:class:`Dataset` types.  This may be extended in the future to contain other schema-like information for non-table datasets.

.. note::
    This differs from the code in ``pipe_supertask`` a bit (other than just being a summary with no docstrings or implementation):
     - I've rewritten ``__init__``'s signature to use ``**kwds`` to allow it to forward all arguments to the ``Task`` constructor.
     - I've removed the ``butler`` argument from ``defineQuanta``; I don't think it's necessary.
     - I've removed ``write_config`` and ``_get(_resource)_config_name``; I think writing is the responsibility of the ``PreflightFramework``, and I think the config name should always be set from ``_DefaultName`` (which is part of ``Task``, not just :py:class:`SuperTask`).
     - Removed ``write_schema`` in favor of ``getDatasetSchemas``.  Again, I think writing should be the responsibility of the ``PreflightFramework``. so we just need a way for it to get the schema(s) from the :py:class:`SuperTask`.

.. _supertask_interface_configuration:

Configuration and DatasetField
------------------------------

The actual :py:class:`Dataset` types used by a :py:class:`SuperTask` are configurable, allowing new types to be defined at configuration time.  The :py:class:`Units <Unit>` utilized by these types are fixed by the concrete :py:class:`SuperTask's <SuperTask>` definition, however, and only the names may be configured.  This will be handled by a new :py:class:`DatasetField` class in ``pex_config`` that is customized for holding :py:class:`Dataset` definitions.


.. _quantum_interface:

Quantum Class Interface
-----------------------

:py:class:`Quantum` is a simple struct-like class that simply aggregates the input and output datasets for a unit of work that can be performed independently by a :py:class:`SuperTask`:

.. py:class:: Quantum

    .. py:attribute:: inputs

        A dictionary of input datasets, with :py:class:`Dataset` types as keys and a `set` of :py:class:`Dataset` instances as values.

    .. py:attribute:: outputs

        A dictionary of output datasets, with the same form as :py:attr:`inputs`


.. _pipeline_interface:

Pipeline Class Interface
========================

.. py:class:: Pipeline

    Pipeline behaves like (at should probably be implemented as) a thin layer over Python's built-in `OrderedDict`, in which the dictionary values hold a concrete :py:class:`SuperTask` subclass and its configuration and the keys are simply string labels.  The order of the items must be consistent with the partial ordering implied by the sequence of :py:class:`Dataset` classes used by the concrete :py:class:`SuperTasks <SuperTask>`, though this is condition is only checked on request -- trying to maintain it as a class invariant would make it much more difficult to modify the Pipeline in-place.

    .. py:method:: checkOrder(self)

        Return False if any :py:class:`SuperTask` in the py:class:`Pipeline` produces an output :py:class:`Dataset` that has already been utilized as an input by a :py:class:`SuperTask` that appears earlier in the :py:class:`Pipeline's <Pipeline>` iteration order.

    .. py:method:: sort(self):

        Modify the iteration order of the :py:class:`Pipeline` to guarantee
        that subsequent calls to :py:meth:`checkOrder` will return True.

    .. py:method:: applyConfigOverrides(self, overrides)

        Apply a set of configuration overrides to the :py:class:`SuperTask` labeled with the given key.  The overrides are given as a dictionary with keys matching labels for :py:class:`SuperTasks <SuperTask>` in the :py:class:`Pipeline`, and values holding configuration overrides for that :py:class:`SuperTask`.

        .. note::
            This assumes a Python class representing a set of config overrides, which ``pex_config`` currently does not provide.


.. _data_id_mapping:

Relating and Specifying Data IDs
================================

The Problem
-----------

The procedure for creating an execution plan for a full :py:class:`Pipeline` reveals some clear limitations in the current `Butler/CmdLineTask` approach to specifying and utilizing dictionary-based data IDs.

As an example, let us consider a :py:class:`SuperTask` responsible for warping a visit-level image to the coordinate system defined by a sky patch prior coaddition.  The quantum in this case is the set of visit-sensor images that overlap the sky patch, and it is quite conceivable that the user would want to specify or constrain (via wildcards) either the outputs (the sky patches for which coadds should be produced) or the inputs (the set of visits to combine), or both.

Given a general wildcard expression that could involve inputs, outputs, or both, and a ``Butler`` API for generating the set of related output data IDs given an input data ID (or vice versa), however, we have no good options for how to expand the wildcards.  If we start by expanding the input wildcard, but the user has only constrained the outputs, we will iterate over all visits in the repository despite the fact that we only need a small fraction of them, and if we start with outputs, the reverse is equally likely.  Whether the wildcard expansion happens within the ``Butler``, in a ``PreflightActivator``, or :py:meth:`SuperTask.defineQuanta`, a way to relate data IDs in a pairwise sense is simply not sufficient.  This is even more evident when we consider the fact that this :py:class:`SuperTask` may be only one i a much larger :py:class:`Pipeline` that involes many other kinds of data IDs that the user may want to constrain.


A Solution: Repository Graphs and Databases
-------------------------------------------

The above problem is not a novel one: it is exactly the problem a relational database's query optimizer attempts to solve when parsing an expression that involves one or more inner joins.  A natural solution in our context is thus to:

 - create a SQL database with a schema that describes the different kinds of data IDs in a repository and their relationships;

 - accept data ID expressions in fhe form of partial SQL where clauses;

 - construct and execute a SELECT query that inner-joins the relevant data IDs and applies the user's data ID expressions.

This represents a complete redesign of the system of managing metadata in a Data Repository.  It replaces the simple, raw-data-centric registry database and the APIs for interacting it with with a multi-table database that manages all datasets in a repository.  To represent the results of the queries against this database in Python, it also involves a replacing the dictionary-based data ID concept with a more object-oriented system that can hold relationship information.  These interfaces are more naturally a part of the Butler Library than the SuperTask Library, and we expect the design sketch described in this section evolve in the course of future Butler Library design work.  However, we do not expect this evolution to require significant changes to the rest of the SuperTask Library design.

In the new system, the combination of a dictionary-style data ID and a dataset type name becomes an instance of the :py:class:`Dataset` class.  A key-value pair in that dictionary becomes an instance of the :py:class:`Unit` class (for "unit of data"); a :py:class:`Dataset` is conceptually a tuple of :py:class:`Units <Unit>`.  A set of :py:class:`Units <Unit>` and py:class:`Datasets <Dataset>` naturally forms a graph-like data structure called a :py:class:`RepoGraph`, which represents (a subset of) a Data Repository.

.. py:class:: Dataset

    A concrete subclass of the abstract base class :py:class:`Dataset` represents a Butler dataset type: a combination of a name, a storage format, path template, and a set of concrete :py:class:`Unit` subclass type objects that define the units of data that label an instance of the dataset.  If, for example, ``Coadd`` is a :py:class:`Dataset` subclass, the corresponding unit classes might be those for :py:class:`Tract`, :py:class:`Patch`, and :py:class:`Filter`.

    An instance of a :py:class:`Dataset` subclass is thus a handle to a particular Butler dataset; it is the only required argument to ``Butler.get`` in the new system, and one of only two required arguments to :py:class:`Butler.put` (the other being the actual object to store).

    :py:class:`Dataset` subclasses are typically created dynamically (usually via a :py:class:DatasetField` that is part of a :py:class:`SuperTask's <SuperTask>` config class).

    .. py:staticmethod:: subclass(name, UnitClasses)

        Define a new :py:class:`Dataset` subclass dyamically with the given name, with instances of the new class required to hold instances of exactly the given :py:class:`Unit` subclasses (via a named attribute for each :py:class:`Unit` subclass).

.. py:class:: Unit

    :py:class:`Unit` is the base of a single-level hierarchy of largely predefined classes that define a static data model.  Each concrete :py:class:`Unit` subclass represents a type of unit of data, such as visits, sensors, or patches of sky, and instances of those classes represent *actual* visits, sensors, or patches of sky.

    A particular :py:class:`Unit's <Unit>` existence is not tied to the presence of any actual data in a repository; it simply defines a dimension in which one or more :py:class:`Datasets <Dataset>` *may* exist.  In addition to fields that describe them (such as a visit number, sensor label, or patch coordinates), concrete :py:class:`Units <Unit>` also have attributes that link them to related :py:class:`Units <Unit>` (such as the set of visit-sensor combinations that overlap a sky patch, and vice versa)

    .. py::attribute:: datasets

        A dictionary containing all :py:class:`Dataset` instances that refer to this :py:class:`Unit` instance.  Keys are :py:class:`Dataset` subclasses, and values are sets of instances of that subclass.

    .. py::attribute:: related

        A dictionary containing all :py:class:`Unit` instances that are directly related to this instance.  Keys are :py:class:`Unit` subclasses, and values are sets fo instances of that subclass.

.. py:class:: RepoGraph

    The attributes that connect :py:class:`Units <Unit>` to other :py:class:`Units <Unit>`, :py:class:`Datasets <Dataset>` to :py:class:`Units <Unit>`, and :py:class:`Units <Unit>` to :py:class:`Datasets <Dataset>` naturally form a graph data structure, which we call a :py:class:`RepoGraph`.

    Because the graph structure is mostly defined by its constituent classes :py:class:`RepoGraph` simply provides flat access to these.

    .. py:attribute:: units

        A dictionary with :py:class:`Unit` subclasses as keys and sets of :py:class:`Unit` instances of that type as values.  Should be considered read-only.

    .. py:attribute:: datasets

        A dictionary with :py:class:`Dataset` subclasses as keys and sets of :py:class:`Dataset` instances of that type as values.  Should be considered read-only.

    .. py::method:: addDataset(self, DatasetClass, **units)

        Create and add a :py:class:`Dataset` instance to the graph, ensuring it is proprely added to the back-reference dictionaries of the :py:class:`Units <Unit>` that define it.  The :py:class:`Dataset` instance is not actually added to the data repository the graph represents; adding them to the graph allows it represent the expected future state of the repository after the processing that produces the dataset has completed.


Connecting Python to SQL
------------------------

The naive approach to mapping these Python classes to a SQL database involves a new table for each :py:class:`Unit` and :py:class:`Dataset` subclass.  It also requires additional join tables for any :py:class:`Units <Unit>` with many-to-many relationships, and probably additional tables to hold camera-specific information for concrete :py:class:`Unit`.  Overall, this approach closely mirrors that of the `Django Project <https://www.djangoproject.com/>`_, in which the custom descriptors that define the attributes of the classes representing database tables can be related directly to the fields of those tables.

The naive approach may work for an implementation based on per-data-repository SQLite databases.  Such an implementation will be important for supporting development work and science users on external systems, but it will not be adequate for most production use cases, which we expect to use centralized database servers to support all repositories in the Data Backbone.  This will require a less-direct mapping between Python classes and SQL tables, especially to avoid the need to permit users to add new tables for new :py:class:`Datasets <Dataset>` when a :py:class:`SuperTask` is run.


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
