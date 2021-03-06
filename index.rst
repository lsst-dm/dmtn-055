
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

This document describes the preliminary design for the SuperTask Library, an abstraction layer and set of utilities that are intended to allow LSST's algorithmic pipeline code to be executed across a broad range of computing environments, usage contexts, and input/output storage systems.
A key goal of SuperTask is to minimize any per-environment per-algorithm customization, allowing a cleaner divide between the responsibilities of algorithm developers, workflow/control system developers, and operators.

The smallest pieces of algorithmic code managed by this system are concrete :py:class:`SuperTasks <SuperTask>`, which inherit from the abstract base class provided by the library.
The library will contain code to combine a set of :py:class:`SuperTasks <SuperTask>` (called a :py:class:`Pipeline`) with a specification of which units of data to process to produce a description of the processing to be done that can be consumed by workflow systems.

Some elements of this design are still unspecified, because they rely heavily on the capabilities and interfaces of the ``Butler`` data access abstraction layer.
The SuperTask Library design necessarily puts new requirements on how ``Butler`` manages and reports relationships between datasets, and a ``Butler`` redesign is currently in progress to address these (and other) requirements.

.. note::

    Once the ``Butler`` redesign is complete and the :py:class:`SuperTask` design has been updated accordingly, this document will be deprecated and its content moved to `LDM-152 <https://ldm-152.lsst.io>`_, the LSST DM Middleware Design Document.


.. _task_config_context:

Task/Config Context
===================

The design for the SuperTask Library sits on top of the existing LSST concepts of *Task* and *Config* classes.
All SuperTasks are Tasks, and utilize the same Config system for algorithmic configuration parameters.

A concrete Task is simply a configurable, composable, callable object.
While all Tasks inherit from a common abstract base class and define a ``run`` method, each Task defines its own signature for ``run``, so Tasks do not really share a common interface.
The ``lsst.pipe.base.Task`` abstract base class itself exists largely to provide utility code to its subclasses, such as setting up logging, providing objects to hold processing metadata, and setting up nested Tasks to be delegated to, called *subtasks*.

This composition of Tasks is closely tied to our approach for configuring them, and it is this functionality that makes the Task concept so useful.
Configuration options for a Task are defined by a corresponding class that inherits from ``lsst.pex.config.Config``.
Config classes contain ``lsst.pex.config.Field`` instances, which act like introspectable properties that define the types, allowed values, and documentation for configuration options.
A set of configuration values for a Task is thus just an instance of its Config class, and overrides for those values can be expressed as Python code that assigns values to to the ``Field`` attributes.
When a Task delegates to another as a subtask, its Config class usually contains a special ``ConfigurableField`` that holds an instance of the subtask's Config class.
``ConfigurableField`` allows this Config instance to be replaced by one for a different Task, allowing the subtask to be replaced by another with the same ``run`` signature.

The :py:class:`SuperTask` abstract base class inherits from ``Task``, and its concrete subclasses are expected to defined a Config class to define their configuration parameters and delegate additional work to subtasks.
Using a SuperTask *as* a subtask is not meaningful, however; in that context the SuperTask just behaves like a regular Task and the additional interfaces and functionality added by SuperTask go unused (as a result, we expect this to be rare).

A few additional properties of Tasks are particularly relevant for SuperTask design:

- The configuration of a Task is frozen after the Task is constructed.

- The schema of any catalogs produced by a Task must be fully defined after Task construction, and must not depend on the actual contents of any data products.

- Calls to ``run`` or any other methods must not change any internal state.


.. _functional_design:

Functional Design and Usage Pattern
===================================

The design of the SuperTask Library is largely derived from the following two principles:

 - Defining units of work that can be performed independently should be a responsibility of the same class (a concrete SuperTask, in this case) that does that work.
Putting this responsibility on the control software or the human user instead would result in a rigid system that is capable of running only a few predefined sequences of SuperTask without requiring significant changes.
While we will likely only need a few predefined sequences late in operations, we need more flexibility during development and early operations.

 - By requesting a list of these units of work from each SuperTask in an ordered list, the control software can discover all dependencies and construct a satisfactory execution plan, in advance, for the full sequence of SuperTasks.
This does not allow the definition of a particular SuperTask's units of work to depend on the actual outputs of an earlier SuperTask in the sequence (as opposed to depending on just the expected presence or absence of outputs, which is supported).

We consider this limitation acceptable for two reasons.
First, we expect cases where the outputs themselves affect the dependencies to be rare, and hence it is an acceptable fallback to simply split the list of SuperTasks into subsets without these dependencies and run the subsets in sequence manually, because the number of such subsets will be small.
More importantly, we believe we can strictly but only slightly overestimate the dependencies between units of work in advance, in essentially all of these cases, and hence the only errors in the execution plan will be a small number of do-nothing jobs and/or unnecessary inputs staged to the local compute environment.
These can easily be handled by any practical workflow system.

For the remainder of this document, we will refer to an independent unit of work performed by a SuperTask (and the list of input and output datasets involved) as a *Quantum*.
An ordered list of SuperTasks (which includes their configuration) is what we call a *Pipeline*.
The control software has many components with different responsibilities, which we will introduce in the remainder of this section.

The typical usage pattern for the SuperTask Library is as follows.

#.  A developer defines a :py:class:`Pipeline` from a sequence of :py:class:`SuperTasks <SuperTask>`, including their configuration, either programmatically or by editing a TBD text-based, human-readable file format.
Other developers may then modify the :py:class:`Pipeline` to modify configuration or insert or delete :py:class:`SuperTasks <SuperTask>`, again via either approach.

#.  An operator passes the :py:class:`Pipeline`, an input data repository, and a Data ID Expression (see :ref:`data_id_mapping`) to a PreFlightFramework.
Different PreFlightFrameworks will be implemented for different contexts.
Some PreFlightFrameworks may provide an interface for making a final round of modifications to the :py:class:`Pipeline` at this stage, but these modifications are not qualitatively different from those in the previous step.

#.  The PreFlightFramework passes the :py:class:`Pipeline`, the input data repository, and the Data ID Expression to a *GraphBuilder* (see :ref:`preflight`), which

    - inspects the :py:class:`Pipeline` to construct a list of all dataset types consumed and/or produced by the :py:class:`Pipeline`;
    - queries the data repository to obtain a *RepoGraph* that contains all datasets of these types that match the given Data ID Expression (see :ref:`data_id_mapping`);
    - calls the :py:meth:`defineQuanta <SuperTask.defineQuanta>` method of each :py:class:`SuperTask` in the :py:class:`Pipeline` in sequence, accumulating a list of all quanta to be executed;
    - constructs the *Quantum Graph* (see :ref:`preflight`), a bipartate directed acyclic graph with quantum vertices linked by the dataset vertices they produce and consume.

#.  The Quantum Graph is passed to an ExecutionFramework, along with additional configuration for how the processing is to be performed (changes in this configuration must not change the outputs of the :py:class:`Pipeline` except to allow intermediate datasets to be elided).
The ExecutionFramework may be the same class as the PreFlightFramework (as in ``lsst.pipe.base.CmdLineTask``, which performs both roles), which makes this step a no-op.
It may also be a completely different class that may be run in an entirely different compute environment (via a serialized Quantum Graph).

#.  The ExecutionFramework creates one or more output data repositories and records in them any repository-wide provenance (such as the :py:class:`Pipeline` configuration or software versions).

#.  The ExecutionFramework walks the Quantum Graph according to the partial ordering it defines, and calls ``runQuantum`` on the appropriate concrete SuperTask for each quantum vertex.
Depending on the activator, the SuperTasks may be run directly in the same compute environment, or submitted to a workflow system for execution elsewhere (probably by translating the generic Quantum Graph to a format specific to a particular workflow system).
In some environments a temporary local data repository containing only the datasets consumed by a particular set of quanta may be created in scratch space to support execution in a context in which the original data repositories are not accessible, with output datasets similarly staged back to the true output data repositories.

.. note::

    The above procedure does not provide a mechanism for adding camera-specific overrides to the configuration.
    I think this has to be part of the :py:class:`Pipeline` interface that's done in the first step, not something done later by PreFlightFrameworks.
    That's especially true if we want to permit Pipelines that aggregate data from multiple cameras; in that case I think we'd need the Pipeline itself to hold the overrides for different cameras in addition to the defaults to avoid spurious provenance issues from having different configurations of the same Pipeline in a repo.
    Given that different cameras might even change the SuperTasks we want in a Pipeline, we may need to make it possible to parameterize all of a Pipeline's definition on different Units of data (not just cameras, but filters).
    I'm sure that's doable, but it is not currently supported by the :py:class:`Pipeline` API in this document.

    We may also be able to avoid that mess just giving up entirely on repository-level provenance.
    Given that we will need more fine-grained provenance ultimately anyway, that may be the best approach.


.. _supertask_interface:

SuperTask Class Interface
=========================

.. py:class:: SuperTask(Task)

    .. py:method:: __init__(self, butler=None, **kwargs)

        All concrete SuperTasks must have the :py:meth:`__init__` signature shown here, in which ``**kwargs`` contains only arguments to be forwarded to ``Task.__init__`` (additional keyword-only arguments are also allowed, as long as they have default values).
        The abstract base class does not use the ``butler`` argument, allowing it to be ``None``, and while concrete SuperTasks may or may not use it, they must accept it even if it is unused.
        This allows the schemas associated with input dataset types and the configuration of preceeding SuperTasks to be loaded and used to complete construction of the SuperTask; a SuperTask should not assume any other datasets are available through the given ``Butler``.
        SuperTasks that do use the ``butler`` argument should also provide an alternate way to provide the schemas and configuration (i.e. additional defaulted keyword arguments) to allow them to be constructed without a ``Butler`` when used as a regular ``Task``.
        This also implies that when a :py:class:`Pipeline` constructs a sequence of SuperTasks, it must ensure the schemas and configuration are recorded at each step, not just at the end.

    .. py:method:: run(self, *args, **kwargs)

        This is the standard entry point for all Tasks, with the signature completely different for each concrete Task.
This should perform the bulk of the SuperTask's algorithmic work, operating on in-memory objects for both arguments and return values, and should not utilize a ``Butler`` or perform any I/O.
In rare cases, a SuperTask for which I/O is an integral component of the algorithm may lack a :py:meth:`run` method, or may have multiple methods to serve the same purpose.
As with other Tasks, the return value should be a ``lsst.pipe.base.Struct`` combining named result objects.

    .. py:method:: defineQuanta(self, repoGraph)

        Called during :ref:`pre-flight <preflight>`, in this method a concrete SuperTask subdivides work into independently-executable units (quanta) and relates the input datasets of these to their output datasets.
        The only argument is a :ref:`RepoGraph <data_id_mapping>` instance, a graph object describing the current state of the relevant subset of the input data repository.
        On return, the ``RepoGraph`` should be modified to additionally contain datasets that will be produced by the SuperTask, reflecting the fact that they will be present in the data repository by the time subsequent SuperTask's in the same :py:class:`Pipeline` are executed.
        The return value should be a list of :py:class:`Quantum` instances.

    .. py:method:: runQuantum(self, quantum, butler)

        This method runs the SuperTask on the given :py:class:`Quantum`, using a ``Butler`` for input and output.
        For most concrete SuperTasks, this should simply use ``Butler.get`` to retrieve inputs, call ``run``, and then use ``Butler.put`` to write outputs.

    .. py:method:: getDatasetClasses(self)

        Called during :ref:`pre-flight <preflight>` (before :py:meth:`defineQuanta`), this method returns the sets of input and output :py:class:`Datasets <Dataset>` classes used by this :py:class:`SuperTask`.
        As long as :ref:`DatasetField <supertask_interface_configuration>` is used to control the :py:class:`Dataset` classes utilized by the :py:class:`SuperTask's <SuperTask>`, the default implementation provided by the :py:class:`SuperTask` base class itself should be sufficient.

    .. py:method:: getDatasetSchemas(self)

        This method returns a dict containing the schemas that correspond to any table-like datasets output by the :py:class:`SuperTask`.
        Dictionary keys are :py:class:`Dataset` types.
        This may be extended in the future to contain other schema-like information for non-table datasets.

.. note::

    This differs from the code in ``pipe_supertask`` a bit):

     - I've rewritten ``__init__``'s signature to use ``**kwds`` to allow it to forward all arguments to the ``Task`` constructor.

     - I've removed the ``butler`` argument from ``defineQuanta``; I don't think it's necessary.

     - I've removed ``write_config`` and ``_get(_resource)_config_name``; I think writing is the responsibility of the PreFlightFramework, and I think the config name should always be set from ``_DefaultName`` (which is part of ``Task``, not just :py:class:`SuperTask`).

     - Removed ``write_schema`` in favor of ``getDatasetSchemas``.  Again, I think writing should be the responsibility of the PreFlightFramework. so we just need a way for it to get the schema(s) from the SuperTask.


.. _supertask_interface_configuration:

Configuration and DatasetField
------------------------------

The actual dataset types used by a SuperTask are configurable, allowing new types to be defined at configuration time.
The units of data utilized by these types are fixed by the concrete SuperTask's definition, however, and only the names may be configured.
This will be handled by a new ``DatasetField`` class in ``pex_config`` that is customized for holding dataset definitions.


.. _quantum_interface:

Quantum Class Interface
-----------------------

:py:class:`Quantum` is a simple struct-like class that simply aggregates the input and output datasets for a unit of work that can be performed independently by a :py:class:`SuperTask`:

.. py:class:: Quantum

    .. py:attribute:: inputs

        A dictionary of input datasets, with dataset type names as keys and a `set` of :py:class:`Dataset` instances as values.

    .. py:attribute:: outputs

        A dictionary of output datasets, with the same form as :py:attr:`inputs`

    .. py:attribute:: task

        The SuperTask instance that produced and should execute this set of inputs and outputs.


.. _pipeline_interface:

Pipeline Class Interface
========================

.. py:class:: Pipeline

    Pipeline behaves like (and should probably be implemented as) a thin layer over Python's built-in `OrderedDict`, in which the dictionary values hold a concrete :py:class:`SuperTask` subclass and its configuration and the keys are simply string labels.
    The order of the items must be consistent with the partial ordering implied by the sequence of :py:class:`Dataset` classes used by the concrete :py:class:`SuperTasks <SuperTask>`, though this is condition is only checked on request -- trying to maintain it as a class invariant would make it much more difficult to modify the Pipeline in-place.

    .. py:method:: checkOrder(self)

        Return False if any :py:class:`SuperTask` in the py:class:`Pipeline` produces an output :py:class:`Dataset` that has already been utilized as an input by a :py:class:`SuperTask` that appears earlier in the :py:class:`Pipeline's <Pipeline>` iteration order.

    .. py:method:: sort(self):

        Modify the iteration order of the :py:class:`Pipeline` to guarantee
        that subsequent calls to :py:meth:`checkOrder` will return True.

    .. py:method:: applyConfigOverrides(self, overrides)

        Apply a set of configuration overrides to the :py:class:`SuperTask` labeled with the given key.
        The overrides are given as a dictionary with keys matching labels for :py:class:`SuperTasks <SuperTask>` in the :py:class:`Pipeline`, and values holding configuration overrides for that :py:class:`SuperTask`.

        .. note::
            This assumes a Python class representing a set of config overrides, which ``pex_config`` currently does not provide.


.. _data_id_mapping:

Relating and Specifying Data IDs
================================

The Problem
-----------

The procedure for creating an execution plan for a full :py:class:`Pipeline` reveals some clear limitations in the current ``Butler``/``CmdLineTask`` approach to specifying and utilizing dictionary-based data IDs.

As an example, let us consider a :py:class:`SuperTask` responsible for warping a visit-level image to the coordinate system defined by a sky patch prior to coaddition.
The quantum in this case is the set of visit-sensor images that overlap the sky patch, and it is quite conceivable that the user would want to specify or constrain (via wildcards) the outputs (the sky patches for which coadds should be produced), the inputs (the set of visits to combine), or both.

Given a general wildcard expression that could involve inputs, outputs, or both, and a ``Butler`` API for generating the set of related output data IDs given an input data ID (or vice versa), however, we have no good options for how to expand the wildcards.
If we start by expanding the input wildcard, but the user has only constrained the outputs, we will iterate over all visits in the repository despite the fact that we only need a small fraction of them, and if we start with outputs, the reverse is equally likely.
Whether the wildcard expansion happens within the ``Butler``, in a PreflightActivator, or in :py:meth:`SuperTask.defineQuanta`, a way to relate data IDs in a pairwise sense is simply not sufficient.
This is even more evident when we consider the fact that this :py:class:`SuperTask` may be only one i a much larger :py:class:`Pipeline` that involes many other kinds of data IDs that the user may want to constrain.


A Solution: Repository Graphs and Databases
-------------------------------------------

The above problem is not a novel one: it is exactly the problem a relational database's query optimizer attempts to solve when parsing an expression that involves one or more inner joins.
A natural solution in our context is thus to:

 - create a SQL database with a schema that describes the different kinds of data IDs in a repository and their relationships;

 - accept data ID expressions in fhe form of partial SQL where clauses;

 - construct and execute a SELECT query that inner-joins the relevant data IDs and applies the user's data ID expressions.

This represents a complete redesign of the system of managing metadata in a Data Repository.
It replaces the simple, raw-data-centric registry database and the APIs for interacting it with a multi-table database that manages all datasets in a repository.
To represent the results of the queries against this database in Python, it also involves a replacing the dictionary-based data ID concept with a more object-oriented system that can hold relationship information.
These interfaces are more naturally a part of the Butler Library than the SuperTask Library, and we expect the design sketch described in this section evolve in the course of future Butler Library design work.
However, we do not expect this evolution to require significant changes to the rest of the SuperTask Library design.

In the new system, the combination of a dictionary-style data ID and a dataset type name becomes an instance of the :py:class:`Dataset` class.
A key-value pair in that dictionary becomes an instance of the :py:class:`Unit` class (for "unit of data"); a :py:class:`Dataset` instance is conceptually a tuple of :py:class:`Units <Unit>`.
A set of :py:class:`Units <Unit>` and :py:class:`Datasets <Dataset>` naturally forms a graph-like data structure called a :py:class:`RepoGraph`, which represents (a subset of) a Data Repository.

.. py:class:: Dataset

    A concrete subclass of the abstract base class :py:class:`Dataset` represents a Butler dataset type: a combination of a name, a storage format, path template, and a set of concrete :py:class:`Unit` subclass type objects that define the units of data that label an instance of the dataset.
    If, for example, ``Coadd`` is a :py:class:`Dataset` subclass, the corresponding unit classes might be those for ``Tract``, ``Patch``, and ``Filter``.

    An instance of a :py:class:`Dataset` subclass is thus a handle to a particular Butler dataset; it is the only required argument to ``Butler.get`` in the new system, and one of only two required arguments to ``Butler.put`` (the other being the actual object to store).

    :py:class:`Dataset` subclasses are typically created dynamically (usually via a :py:class:DatasetField` that is part of a :py:class:`SuperTask's <SuperTask>` config class).

    .. py:staticmethod:: subclass(name, UnitClasses)

        Define a new :py:class:`Dataset` subclass dynamically with the given name, with instances of the new class required to hold instances of exactly the given :py:class:`Unit` subclasses (via a named attribute for each :py:class:`Unit` subclass).

    .. py:attribute:: units

        A dictionary containing the units that identify this dataset, with unit type names as keys and :py:class:`Unit` instances as values.

    .. py:attribute:: creator

        Optional.  A pointer to the :py:class:`Quantum` object that represents the processing steps that (will) produce this dataset.

    .. py:attribute:: consumers

        A (possibly empty) set of :py:class:`Quantum` objects that represent the processing steps that use this dataset as an input.

.. py:class:: Unit

    :py:class:`Unit` is the base of a single-level hierarchy of largely predefined classes that define a static data model.
    Each concrete :py:class:`Unit` subclass represents a type of unit of data, such as visits, sensors, or patches of sky, and instances of those classes represent *actual* visits, sensors, or patches of sky.

    A particular :py:class:`Unit's <Unit>` existence is not tied to the presence of any actual data in a repository; it simply defines a dimension in which one or more :py:class:`Datasets <Dataset>` *may* exist.
    In addition to fields that describe them (such as a visit number, sensor label, or patch coordinates), concrete :py:class:`Units <Unit>` also have attributes that link them to related :py:class:`Units <Unit>` (such as the set of visit-sensor combinations that overlap a sky patch, and vice versa)

    .. py:attribute:: datasets

        A dictionary containing all :py:class:`Dataset` instances that refer to this :py:class:`Unit` instance.
        Keys are dataset type names, and values are sets of instances of that subclass.

    .. py:attribute:: related

        A dictionary containing all :py:class:`Unit` instances that are directly related to this instance.
        Keys are unit type names, and values are sets fo instances of that subclass.

.. py:class:: RepoGraph

    The attributes that connect :py:class:`Units <Unit>` to other :py:class:`Units <Unit>`, :py:class:`Datasets <Dataset>` to :py:class:`Units <Unit>`, and :py:class:`Units <Unit>` to :py:class:`Datasets <Dataset>` naturally form a graph data structure, which we call a :py:class:`RepoGraph`.

    Because the graph structure is mostly defined by its constituent classes :py:class:`RepoGraph` simply provides flat access to these.

    .. py:attribute:: units

        A dictionary with unit type names as keys and sets of :py:class:`Unit` instances of that type as values.
        Should be considered read-only.

    .. py:attribute:: datasets

        A dictionary with dataset type names as keys and sets of :py:class:`Dataset` instances of that type as values.
        Should be considered read-only.

    .. py:method:: addDataset(self, DatasetClass, **units)

        Create and add a :py:class:`Dataset` instance to the graph, ensuring it is proprely added to the back-reference dictionaries of the :py:class:`Units <Unit>` that define it.
        The :py:class:`Dataset` instance is not actually added to the data repository the graph represents; adding them to the graph allows it represent the expected future state of the repository after the processing that produces the dataset has completed.

.. py:function:: makeRepoGraph(repository, NeededDatasets, FutureDatasets, where)

    Construct a :py:class:`RepoGraph` representing a subset of the given data repository by executing a SQL query against the repository database and interpreting the results.

    :param str repository: a string URI identifying the input data repository.

    :param tuple NeededDatasets: a tuple of :py:class:`Dataset` subclass type objects whose instances and corresponding :py:class:`Units <Unit>` must be included in the graph, and restricted to only datasets already present in the input data repository.

    :param tuple FutureDatasets: a tuple of :py:class:`Dataset` subclass type objects whose :py:class:`Unit <Unit>` types must be included in the graph, but whose instances should not not be restricted by what is present in the data repository.

    :param str where: a string containing a SQL ``WHERE`` clause against the schema defined by the set of :py:class:`Unit` classes in the repository, which will be used to restrict the :py:class:`Units <Unit>` and :py:class:`Datasets <Dataset>` in the returned graph.

    :return: a :py:class:`RepoGraph`

    Like other interfaces that interact with a data repository, this function may ultimately become part of a Butler API (with the ``repository`` argument removed, as the Butler would then be initialized with that repository).


Connecting Python to SQL
------------------------

The naive approach to mapping these Python classes to a SQL database involves a new table for each :py:class:`Unit` and :py:class:`Dataset` subclass.
It also requires additional join tables for any :py:class:`Units <Unit>` with many-to-many relationships, and probably additional tables to hold camera-specific information for concrete :py:class:`Unit`.
Overall, this approach closely mirrors that of the `Django Project <https://www.djangoproject.com/>`_, in which the custom descriptors that define the attributes of the classes representing database tables can be related directly to the fields of those tables.

The naive approach may work for an implementation based on per-data-repository SQLite databases.
Such an implementation will be important for supporting development work and science users on external systems, but it will not be adequate for most production use cases, which we expect to use centralized database servers to support all repositories in the Data Backbone.
This will require a less-direct mapping between Python classes and SQL tables, especially to avoid the need to permit users to add new tables for new :py:class:`Datasets <Dataset>` types when a :py:class:`SuperTask` is run.


.. _preflight:

Pre-Flight Environment
======================

With the class interfaces described in the last few sections, we can now more fully describe the "pre-flight" procedure summarized in Section :ref:`functional_design`.
Unlike the :ref:`quantum execution environment <quantum_execution>`, most of preflight is common code shared by all PreFlightFrameworks, which simply provide different front-end APIs appropriate for their users and supply an appropriate implementation of :py:func:`makeRepoGraph` for the given input data repository.

The inputs to all PreFlightFrameworks (though one or more may be defaulted) are:

 - The input data repository or a Butler initialized to point to it.

 - A user expression defining the units of data to process, in the form of a SQL ``WHERE`` clause that can be passed *directly* to :py:func:`makeRepoGraph`.

 - A :py:class:`Pipeline` instance.

A PreFlightFramework delegates essentially all remaining work to the :py:class:`QuantumGraphBuilder`:

 - The PreFlightFramework constructs a :py:class:`QuantumGraphBuilder`, passing it the :py:class:`Pipeline` instance.
The :py:class:`QuantumGraphBuilder` instantiates all SuperTasks in the :py:class:`Pipeline`, collecting their (now frozen) configuration, schemas, and input and output dataset types.

 - The PreFlightFramework creates a :py:class:`RepoGraph` from the input data repository, the user ``WHERE`` expression, and the lists of dataset types reported by the :py:class:`QuantumGraphBuilder` by calling `py:func:`makeRepoGraph`.
The design also leaves open the possibility that the operations PreFlightFramework will construct a :py:class:`RepoGraph` by some other means (which could support a more complicated set of SQL queries that target an operations-specific SQL schema).

 - :py:meth:`QuantumGraphBuilder.makeGraph` is called with the :py:class:`RepoGraph` to build the Quantum Graph.

.. note::

    This differs from the code in ``pipe_supertask`` in two big ways:

     - I've renamed the class from ``GraphBuilder`` to ``QuantumGraphBuilder`` for better disambiguation with ``makeRepoGraph``.

     - I've switched up the construction and ``makeGraph`` arguments, which allows us to generate the :py:class:`RepoGraph` separately, which may be necessary to address some operations concerns.  I don't think that we gained anything from initializing ``GraphBuilder`` with the repository and the user expression in the old design.

A more detailed description of :py:class:`QuantumGraphBuilder` is below.

.. py:class:: QuantumGraphBuilder

    .. py:method:: __init__(self, pipeline, butler)

        The :py:class:GraphBuilder` first iterates over the SuperTasks in the :py:class:`Pipeline`, instantiating them (which freezes their configuration), and accumulating a list of input and output dataset types by calling :py:meth:`SuperTask.getDatasetClasses` on each.
        Dictionaries containing configuration and schemas are also constructed for later use in recording provenance.

        .. note::

            While instantiating a SuperTask in general requires a Butler, this is mostly to allow downstream SuperTasks to obtain the schemas of their input dataset types.
            While there's no way to avoid having :py:class:`QuantumGraphBuilder` use the given Butler to load the schemas of the overall input dataset types (assuming any of these are catalogs), it could use a dummy Butler backed by a simple dict to transfer schemas obtained by calling :py:meth:`SuperTask.getDatasetSchemas()` to downstream :py:meth:`SuperTask.__init__`.
            At the same time, it would build up its own py:attr:`schemas` attribute, which could be used by the PreFlightFramework to actually persist the schemas.

    .. py:attribute:: NeededDatasets

        A ``set`` of dataset types (subclasses of :py:class:`Dataset`) that are used strictly as inputs by the :py:class:`Pipeline` the :py:class:`QuantumGraphBuilder` was constructed with.

    .. py:attribute:: FutureDatasets

        A ``set`` of dataset types (subclasses of :py:class:`Dataset`) that are produced as outputs (including intermediates) by the :py:class:`Pipeline` the :py:class:`QuantumGraphBuilder` was constructed with.

    .. py:attribute:: configs

        A ``dict`` mapping SuperTask name to Config instance.

    .. py::attribute:: schemas

        A ``dict`` mapping :py:class:`Dataset` subclass to :py:class:`lsst.afw.table.Schema`, with entries only for output catalog datasets.

    .. py:method:: makeGraph(self, repoGraph)

        Construct a :py:class:`QuantumGraph` representing (conceptually) the processing to be performed and its dependencies.

        This is implemented by iterating through the SuperTasks instantiated by :py:meth:`__init__`, calling :py:meth:`SuperTask.defineQuanta` with the :py:class:`RepoGraph`.
As each SuperTask defines its quanta, it also adds the :py:class:`Datasets <Dataset>` it will produce to the :py:class:`RepoGraph`, making it appear to subsequent SuperTasks that these datasets are already present in the repository and may be used as inputs.
The result of this iteration is a sequence of :py:class:`Quantum` instances.

        The final step is to transfrom this sequence into the Quantum Graph, which is a directed acyclic graph describing the dependencies of the processing.
        Each node in the Quantum Graph is conceptually either a :py:class:`Quantum` or a :py:class:`Dataset`, with the direction of the graph edges representing inputs (:py:class:`Dataset` node to :py:class:`Quantum` node) and outputs (:py:class:`Quantum` node to :py:class:`Dataset` node).
        Because each :py:class:`Quantum` instance holds its input and output :py:class:`Dataset` instances, the only remaining step to making the sequence of quanta into a fully-walkable graph is to add back-references from each :py:class:`Dataset`, filling in its :py:attr:`creator <Dataset.creator>` and :py:attr:`consumers <Dataset.consumers>` attributes to point to the appropriate :py:class:`Quantum` instances.


.. py:class:: QuantumGraph

    The attributes that connect :py:class:`Quanta <Quantum>` to :py:class:`Datasets <Dataset>` naturally form a graph data structure, which we call a :py:class:`QuantumGraph`.

    Because the graph structure is mostly defined by its constituent classes, :py:class:`QuantumGraph` simply provides flat access to these.

    .. py:attribute:: quanta

        A list of :py:class:`Quantum` instances, ordered in a way that satisfies dependencies (which may be unique).

    .. py:attribute:: datasets

        A dictionary with dataset type names as keys and sets of :py:class:`Dataset` instances of that type as values.


.. _quantum_execution:

Quantum-Execution Environment
=============================

Unlike the pre-flight environment, the code that implements the quantum execution environment in which :py:meth:`SuperTask.runQuantum` is called and actual algorithmic code is run is in general not shared between different implementations.

A QuantumExecutionFramework can be as simple as a thin layer that provides a call to :py:meth:`SuperTask.runQuantum` with a Butler or as complex as a multi-level workflow system that involves staging data to local filesystems, strict provenance control, multiple batch submissions, and automatic retries.

At the lowest level, all QuantumExecutionFrameworks will have to do at least the following tasks:

 - Instantiate one or more SuperTasks from the :py:class:`Pipeline` (ensuring that this is done consistently with how they were instantiated in pre-flight).  This also involves initializing logging for SuperTask(s) and their subtasks, and will require setting up a Butler (possibly a simple dict-backed one) to facilitate the transfer of schema information.

 - Create a Butler (possibly the same as the one used for SuperTask construction).

 - Call :py:meth:`SuperTask.runQuantum` on each of the :py:class:`Quantum` instances it is responsible for running.

When careful control over provenance is necessary, the Butler passed to :py:meth:`SuperTask.runQuantum` can be instrumented to detect the actual datasets loaded by the task, though even this probably cannot fully replace reporting by the task itself about what was used.

When data is staged to a local filesystem for execution, the Butler created in the local filesystem only needs to provide the capability to ``get`` and ``put`` the input and output datasets that are included in the quanta to be executed.
Because the mappings between the :py:class:`Datasets <Dataset>` in the quanta and the staged files can be fully determined at pre-flight, the Butler implementation here can be quite simple as long as the staging system can transfer an additional file containing those mappings.


.. _implementations:

Notes on specific expected implementations
==========================================


.. _command_line_implementation:

CmdLineFramework
----------------

The only complete framework for running SuperTasks that will be provided by the SuperTask Library itself is CmdLineFramework, which provides both a PreFlightFramework that can be used from command-line shells and a QuantumExecutionFramework that runs :py:class:`Pipeline` in a single-node, multi-core environment using Python's ``multiprocessing`` module.

The QuantumExecutionFramework provided by CmdLineActivator will also have programmatic entry points, which may permit it to be used by batch-based execution frameworks for running a subset of a larger job on a particular node.


.. _operations_batch_implementation:

Data Release Production in Operations
-------------------------------------

The batch system used to produce LSST data releases (as well as large-scale integration tests) will execute SuperTasks using a workflow system probably based on either Pegasus/HTCondor or the DES Data Management system (which also uses HTCondor), with persistent storage provided by the Data Backbone and metadata information provided by a monolithic database server.
Whenever possible, jobs will use local temporary storage rather than a global filesystem, and SuperTasks executed by this system will never write to the Data Backbone directly.

Because (at least) the vast majority of LSST DRP processing can be split into completely independent spatial tiles, we expect to process each of these tiles (which may involve multiple skymap "tracts") manually in turn, rather than have the workflow system attempt to schedule the entire production with a single invocation; this should drastically decrease the amount of storage needed for intermediate data products.
Similarly, while the Science Pipelines team may deliver a single :py:class:`Pipeline` that represents the entirety of DRP processing, operators may choose to split this into multiple smaller :py:class:`Pipeline <Pipeline>` that can be run manually in serial over a sky tile.

A major challenge in developing the SuperTask execution framework for operations batch processing is reconciling the database system needed by the operations system to manage metadata, detailed provenance information and fine-grained control over input data products with the data repository and "simple common schema" concepts that form the core of the system described in :ref:`data_id_mapping`.
The best possible outcome of the remaining design work in this area is that we find a way to define the common schema containing :py:class:`Units <Unit>` and :py:class:`Datasets <Dataset>` as a set of SQL views on the full operations schema, and that this interface, when fully fleshed-out, provides sufficient fine-grained control to meet the needs of operations.
As an intermediate fallback, operators could write queries to select input data and units of processing directly against the full operations schema, while making sure the results of those queries take a form that allows them to be translated into a :py:class:`RepoGraph` via a different piece of code.
In the worst-case scenario, operators would essentially re-implement much of the logic contained in the actual DRP :py:class:`Pipeline's <Pipeline>` :py:meth:`SuperTask.defineQuanta` methods, and use a completely different system to build the processing DAG.
Even in this scenario, the rest of the SuperTask interface could be used as defined here, and :py:class:`SuperTask.defineQuanta` could still be used in other execution contexts.


.. _external_batch_implementation:

Mid-Scale and External Batch Processing
---------------------------------------

A batch-based execution system that does not depend on the full operations environment is important for development during construction and operations (as DM developers and operations scientists are not expected to use the operations system directly), external collaborators (such as the Hyper Suprime-Cam team or the LSST Dark Energy Science Collaboration), and possibly batch execution for science users running in the LSST Science Platform (see :ref:`science_platform_implementation`).

Ideally, this would at least make heavy use of components developed for the batch operations system :ref:`operations_batch_implementation`; this component is, in essence, a version of that system with stronger (but nevertheless vague) requirements on ease-of-use and install and weaker (but also vague) requirements on scalability and robustness.

The combination of a tight schedule for development of the operations system and the lack of clear requirements and responsibility for the development of the external batch system may make reusing many components from the operations system difficult.


.. _science_platform_implementation:

LSST Science Platform
---------------------

In the notebook environment of the Science Platform, SuperTask will be executed in any of three ways:

 - directly in the Python kernel of the notebook, with no pre-flight (and probably just as regular Tasks via ``Task.run`` rather than :py:meth:`SuperTask.runQuantum`).;

 - in the notebook container, using multiple processes in a manner very similar to (and probably implemented by :ref:`CmdLineFramework <command_line_implementation>`, though we will provide an interface for launching such jobs directly from the notebook);

 - via a batch system attached to the Science Platform, using either a copy or walled-off corner of the :ref:`operations batch system <operations_batch_implementation>`, a version of the :ref:`mid-scale/external system <external_batch_implementation>`, or something more closely-related to Qserv next-to-database processing.

The portal environment of the Science Platform may also launch certain predefined SuperTasks to perform specific image-processing tasks.

.. note::

    I don't really have any sense for what kinds of SuperTasks the portal might want to launch or whether they'd be more appropriately run in a container or in a batch system (though I expect the latter would be rather high-latency for the portal).


.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :encoding: latex+latin
..    :style: lsst_aa
