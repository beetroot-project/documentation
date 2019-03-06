Beetroot project is a set of CMake macros that set a new, user friendly paradigm of writing CMake code. It targets medium to large CMake deployments and helps by faciliating modularization and parametrization of the project description.

The idea was started in 2017, and was initially motivated by a) the fact, that usually CMake scripts rely heavily on global variables and b) CMake has very poor function parameter handling. This led to an effort to define a set of macros that facilate checking of types of function arguments. After several iterations and complete redesigning, the current shape Beetroot was formed. 

Beetroot tries to nudge developers to put their targets definitions and dependency declarations inside CMake functions with clear API interface, so it is clear on what information each target depends. In return 

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
