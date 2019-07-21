Overview
=========

Beetroot project is a set of CMake macros that set a new, user friendly paradigm of writing CMake code. It targets medium to large CMake deployments and helps by faciliating modularization and parametrization of the project description.

The idea was started in 2017, and was initially motivated by a) the fact, that usually CMake scripts rely heavily on global variables and b) CMake has very poor function parameter handling. This led to an effort to define a set of macros that facilate checking of types of function arguments. After several iterations and several complete redesignings, the current shape of the Beetroot was formed. 

Beetroot tries to nudge developers to put their targets definitions and dependency declarations inside CMake functions with a clear API interface, so it is clear on what information each target depends. In return 

#. it does a great deal of semantic checks on the user code, 
#. allows a lot of flexibility of where and how to put the user CMake code, 
#. allows to build any part of the project from anywhere,
#. can automatically turn a project into a superbuild if any of the targets are external.

Steps to start using the Beetroot:
==================================

#. Put the file ``root.cmake`` in the subfolder ``cmake`` of the root of your project. The file name and path is not hard-coded and can easily be changed. It is there so the Beetroot can be aware where is the root of your project.
#. Put a standard common header in the beginning of the ``CMakeLists.txt``, add a call ``finalizer()`` at the end of it.
#. Write target description files (either ``targets.cmake`` anywhere in the project tree, or any file with the extension ``.cmake`` in the subdirectory ``cmake/targets`` anywhere in the project tree.
#. Refer to the target you want to build inside the ``CMakeLists.txt`` by function ``build_target(<TEMPLATE_NAME> [<ARGUMENTS>])``



Why Beetroot?
^^^^^^^^^^^^^

There are many CMake projects that superficially look similar in scope to us. Beetroot is unique in that it aims to replace/deprecate as little of the native CMake commands and idioms as possible, while delivering modern (if not experimental) approach to the build. 

Here is a list of known to me at the time of writing conversion projects with its own description.

* https://github.com/ecmwf/ecbuild - A CMake-based build system, consisting of a collection of CMake macros and functions that ease the managing of software build systems
* https://github.com/DevSolar/jaws - Just A Working Setup.

All these projects are "total conversion mods" for CMake - they replace/reimplement bulk of the CMake functions. In order to use them, you need to learn many new commands with their parameters. Although the new systems are arguably better, you still, as a user, will inevitably learn the standard CMake commands anaway. That will happen either from necessity, because your use case was not forseen and you had to implement it in "bare" CMake, or the standard commands are better documented & supported. This may lead to the situation, contrary to the projects' claims, that using the mentioned systems would lead to the situation that requires you to actually learn more CMake, not less[#fs1]_ . 

When implementing you project in Beetroot you will use most of the CMake commands as you did before. In particular, you still use ``add_library()``, ``add_executable()`` and ``add_custom_target()`` to define your targets. You will access and set the targets' properites with usual ``target_compile_definitions()``, ``target_compile_options()``, ``target_include_directories()`` and even ``target_sources()``. You may use ``target_link_libraries()`` or you may defer to the Beetroot calling it automatically for you. You may use ``ExternalProject`` and ``find_package()`` or you may use the Beetroot's built-in system of handling them. The only command that is deprecated is ``add_subdirectory()`` - it is replaced with a far more controllable and less error-prone mechanism that is used to resolve information-dependency on the various parts of the CMake project.


.. [#fs1] For a humoristic view on this phenomenon take a look at this brilliant XKCD comic strip: https://xkcd.com/927/

There list will not be complete without the mention of the Artichoke: https://github.com/commontk/Artichoke - "CMake module allowing to easily create a build system on top of ExternalProjects." The project adheres to the "do one thing, but do it well" Unix principle, and should be, at least partially, compatible with the Beetroot.


Beetroot data model
===================

Since you, the reader, is most probably a seasoned programmer, I believe the most straightforward way of introducing you to the Beetroot is through data model.

CMake (among other things) maintain internally an object for each defined target, and at the glance, the Beetroot's equivalent to the CMake target is a "`FEATUREBASE`" object class [1]. `FEATUREBASE` also gets constructed if the user's code does not result in CMake targets (because e.g. it only modifies existing target). Also note that one target definition file (`targets.cmake`) can define multiple FeatureBase objects, because user can put several names in the `ENUM_TEMPLATES` stanza. 

Each invocation of a function the requires a build of the target (e.g. `build_target()` or `get_existing_target()`) is internally represented by the object of the type "`INSTANCE`". In Beetroot INSTANCE and FEATUREBASE have one-to-many relationship; each INSTANCE is matched with a single FEATUREBASE, but a single FEATUREBASE may by linked to several INSTANCES.

In plain CMake, this duality does not exist, the sequence of calls `add_library(foo ...)` and `target_link_libraries(myexec PRIVATE foo)` mean in simple English "build a library `foo`, and then make `myexec` depend on it". The `mytarget` alwyas gets defined or an error produced. On the other hand, in Beetroot one may express dependency relationship as "I need a library foo with a param `FLAG`". This would result in a new copy of library `foo` only if it has not been defined alsewhere with the sole param `FLAG`. It would be more equivalent to the 


   if(NOT TARGET foo)
   	add_library(foo ...)
   endif()
   target_link_libraries(myexec PRIVATE foo)

except that such code can be dangerous, if we may not be sure with what options the `foo` was built. Beetroot takes care of that, by matching all the parameters `...` between the pre-existing `foo` and the `foo` definition.

Beetroot also allows for more advanced dependency statement, expressed as "I need whatever library `foo` the project already comes with. Just make sure that among other parameters, it is compiled with a feature indicated by the param FLAG on". In that case Beetroot would gather all requested versions of the library `foo`, decide which are compatible with the specification, then decide if the feature `FLAG` can be added. If not - build a separate copy of `foo`.

If user requires two targets but the only way they differ is through the link parameters, than it is not required to actually build two copies of them and in such case each `INSTANCE` link to the same `FEATUREBASE`:


   build_target(MYTARGET)
   build_target(MYTARGET MYLINKPAR other)

.. image:: 1TARGET_2INSTANCES.png
  :width: 700
  :alt: Build tree of `build_target(MYTARGET MYLINKPAR other); build_target(MYTARGET)`. Orange is `FEATUREBASE` with first title row showing target name and its internal ID. Blue is `INSTANCE` with title row showing its internal ID.



One of the base features of the Beetroot is the ability to build several copies of a target, by simply requiring it with different parameters. If such requirements are mutually incompatible (as is always the case if target parameters differ, but usually not if the features differ, and never with link parameters) than Beetroot will decide to instantiate two distinct FEATUREBASE (and CMake targets) and we will end up with 


   build_target(MYTARGET)
   build_target(MYTARGET PAR 42)


.. image:: 2TARGETS_2INSTANCES.png
  :width: 700
  :alt: Build tree of `build_target(MYTARGET PAR 42); build_target(MYTARGET)`. Orange is `FEATUREBASE` with first title row showing target name and its internal ID. Blue is `INSTANCE` with title row showing its internal ID.

Because one-to-one relationship between an instance and a target is common, it will be later on depicted with a common box like this:

.. image:: 2TARGETS_2INSTANCES_compact.png
  :width: 700
  :alt: Compact (and default) version of the build tree of `build_target(MYTARGET PAR 42); build_target(MYTARGET)`. Orange is `FEATUREBASE` with first title row showing target name and its internal ID. Blue is `INSTANCE` with title row showing its internal ID.

Dependencies between targets are realized as directed links between `INSTANCES`, like this:


.. image:: DEPENDENCY.png
  :width: 700
  :alt: Build tree of `MYEXEC` that depends on `MYLIB`. The dependency relation is always realized between `INSTANCES`, not `FEATUREBASES`.

.. image:: DEPENDENCY_compact.png
  :width: 700
  :alt: Compact view of a tree where `MYEXEC` depends on `MYLIB`.



How does the Beetroot work?
===========================

Initialization
^^^^^^^^^^^^^^

At the beginning, when Beetroot is loaded, it scans all the subfolders of the project to find target definition files and build a database that maps template/target names to the path of the target definition file.

It also initializes internal variables (held inside global CMake storage) and loads all internal functions.

Target declaration phase
^^^^^^^^^^^^^^^^^^^^^^^^

When the initialization is complete, it reads through the rest of the ``CMakeLists.txt`` and expects to find calls to ``build_target(<TEMPLATE_NAME> [<ARGUMENTS>])``. Each call ultimately triggers user defined function ``declare_dependencies()``, where the Beetroot expects to find additional ``build_target()`` calls and marks the target to be defined later on, because no targets will be defined until the call to the ``finalize()`` at the end of the ``CMakeList.txt``. It calls all encountered ``build_target()`` recursively.

Target definition phase
^^^^^^^^^^^^^^^^^^^^^^^

Target definition phase is handled by the call to ``finalize()`` and this is when targets get defined. 

First of all, Beetroot tries to fully declare all targets that were declared with ``build_existing_target()``. 

Once all targets are declared then Beetroot can finally decide whether it is going to do a super build, or project build.

After that, if it is a project build, it enables all declared languages for all targets in the current build tree.

Finally it defines and links all the relevant targets, by calling ``generate_targets()`` user function and then ``apply_dependency_to_target()`` user function and/or ``target_link_libraries()`` CMake built in function. When on superbuild it will only attempt to define external targets.

``finalize()`` returns and by default this should be the end of the ``CMakeLists.txt``.

.. [1] Even though the CMake DSL is not object-oriented, the structure of the Beetroot code most certainly is. The code *simulates* OO features that CMake is missing using various tricks, which are a implementation detail and should not be of concern to the user.
