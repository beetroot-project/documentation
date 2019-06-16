Frequently Asked Questions
==========================

How does the system mix with the plain CMake?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Beetroot-managed targets can be interact with the "legacy" CMake code, if you want to use each target as either dependency, or the dependees - but not both. 

First thing to be aware of is that the Beetroot by default can automatically decide to employ the superbuild idiom. In this case the ``CMakeLists.txt`` will be run *twice* in a single build - once, directly called by you when you type ``cmake ..`` or simmilar, and second time, when you type ``make``, in the so-called project build. So unless you are sure there will be no superbuild run, you should desing your code in such a way, that the targets are defined only in the proper phase, usually in the project build, unless they define external projects. To tell if the current phase is superbuild or project build use a boolean variable ``IS_SUPERBUILD``.

Another issue is the existance of calls to the global target-modifying functions, such as ``??``. If you need to use them, just remember that they still affect the whole project including the beetroot-managed code, otherwise you are fine. Sincerelly, the best thing to do is to hunt them down using "find" method in your code editor and turn them into the target-specific equivalents.

Usage of the "legacy" target as a dependency
---------------------------------------------

Targets defined in the ``CMakeLists.txt`` will be accessible to the beetroot-managed code as the possible dependency *if they are defined before the call to the ``finalize()``*. You would then need to link with those targets manually in the body of the function ``apply_dependency_to_target()``.

"Legacy" target depends on the target(s) managed by the Beetroot 
----------------------------------------------------------------

Beetroot can designate the name of the managed targets itself, so if the "legacy" targets need to depend on the targets governed by the Beetroot, you need to either know those names because you defined them as being fixed, or dynamically get them. You get the name of the targets using the function ``get_target_name()``. Beetroot targets will not get defined until you call the ``finalize()`` function, so write any code depending on them after the call to that function.

What does it take to turn a "legacy" CMake code into being managed by the Beetroot?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

First of you need to isolate the code that defines each target, and put it into a separate ``targets.cmake``. In case if that would lead to a significant code duplication, you have an option to define several targets in one ``targets.cmake`` file. 

Inside the ``targets.cmake`` you need to separate the code into two groups and put them accordingly to its purpose:

#. The code that defines the targets and its properties (such as a call to the  ``add_library()`` or ``target_compile_definitions()``) and does not mention any dependee - put into a definition of ``generate_targets()``. 

#. The code that mentions the dependee. If it is just a call to the ``target_link_libraries(DEPENDEE_TARGET OUR_TARGET)`` you can simply remove it, since Beetroot would call it by itself. If it involves something more unusual, like calls to ``add_custom_command()`` and  ``target_sources()`` (common pattern when you have a code generator) - put that code into an ``apply_dependency_to_target()``. 

 
 
Is there any unexpected way the Beetroot can interefere with my own CMake code?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

CMake does not support namespaces, so however unlikely, it is still possible that your own code can interfere with the internal workings of the Beetroot project. To stay clear of trouble you should remember that all internal Beetroot variables are prefixed with double underscore, and all functions with a single underscore. In future, they will have even more unique prefix.
