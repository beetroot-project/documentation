Frequently Asked Questions
==========================

Q: How does the system mix with the plain CMake? What if I define some targets directly in ``CMakeLists.txt``?
A: No problem. 

How does the system mix with the plain CMake?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

First of all, remember that the ``CMakeLists.txt`` can be run twice - once for superbuild, and then second time for the project build. So unless you are sure there will be no superbuild run, you should desing your code in such a way, that the targets are defined only in the proper phase (usually in the project build, unless they define external projects). To tell if the current phase is superbuild or project build use a boolean variable ``IS_SUPERBUILD``.

Targets defined in ``CMakeLists.txt`` targets will work.

If they need to depend on the targets governed by the Beetroot you need to remember two things. 
#. Is to know or get the name of the target. If the target names are fixed, you simply use that name, as usually. If targets are not fixed you need to get the name of the targets using the functions ``get_target()`` or ``get_existing_target()``. 
#. Beetroot targets will not get defined until you call the ``finalize()`` function, so write any code depending on them after the call to that function.

If you are in a reverse situation - you manually defined target which is a dependency of target(s) inside the Beetroot, you need to
1. Write the dependency code (usually ``target_link_libraries``) inside the ``apply_dependency_to_target()``. If you need to write this function, make sure you put flag ``LINK_TO_DEPENDEE`` inside ``TEMPLATE_OPTIONS`` to make sure you don't interfere with the linking process. 
2. Make sure you define the targets *before* the call to ``finalize()``.


 
 
 Q: How exactly Beetroot can interefere with my own CMake code?
 A: Beetroot goes to great lengths to make sure it will not interfere with any other CMake code. It is still possible (although unlikely) to interfere with the Beetroot project. In particular you should not:
 #. Call Beetroot code (in particular ``finalize``) after you (re)defined functions that have the same name as the ones that are used by the Beetroot. All internal Beetroot functions start with the underscore.
 #. Use any of the reserved global properties. All global properties that the Beetroot uses start with double underscore. 
 
