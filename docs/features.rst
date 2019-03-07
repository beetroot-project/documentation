List of most important and complex Beetroot functionalities
===========================================================

Advanced and convenient handling of targets' parameters
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Parameters to each target are handled in three ways:

1. Default values are specified in the ``targets.cmake``, when declaring the parameters.
2. Default values are overriden by existing variables that have the same name.
3. Existing variables are overriden by named arguments given to the ``get_target`` and ``build_target`` functions.

Furthermore, system allows for default values to include references to the any variables (including defined parameters themselves) and any other CMake string processing commands. For instance, this is a perfectly valid definition::

	get_filename_component(TMP "${ARCH_PREFIX}" NAME)

	set(DEFINE_MODIFIERS 
		ARCH	SCALAR	CHOICE(GPU;CPU) CPU
		ARCH_PREFIX	SCALAR	STRING "${SUPERBUILD_ROOT}/external-${ARCH}"
		ARCH_RELDIR	SCALAR	STRING	"${TMP}"
	)

The intention is to let the user specify only the ``ARCH`` parameter, and the ``ARCH_PREFIX`` and ``ARCH_RELDIR`` gets calculated from it.


Distinction between target parameters and link parameters
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Generally all declared template parameters change the identity of the target. Occasionally though, when the target is used as a dependency, some settings should not change its identity target and trigger a separate build. Those settings include preprocessor macros that are applied for header-only libraries, or are consumed only by the library headers. In this case we can define those parameters as ``LINK_PARAMETERS``. User cannot use them inside the ``generate_target()`` (unless the ``GENERATE_TARGETS_INCLUDE_LINKPARS`` flag is set, and even then they should not be used in target definitions, but for non-target task, such as defining tests) and ``declare_dependencies()`` functions, but they are available in ``apply_to_target()``. 


Target definition can make multiple physical targets, each one with different set of parameters
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

When you require a target ``FOO`` and specify target parameters, the target with this specific set of parameters will be defined and build. If you require the same target ``FOO`` with different set of parameters, by default Beetroot will define *another* instance of the same target, built using the other set of parameters. The targets' CMake names (and file names) will have a numeral suffix to distinguish them apart. This implies that in general you cannot know the name of the target, until you query for it (or you declared that there can be only one instance of it). 


Special handling for target parameters that describe features
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Sometimes a parameter defines *additional feature* of the target, that is turned on/advanced independently of the rest of the target functionality. Parameter is a feature, when the dependee who does not require it can still use the target that has it. Features can be flags like ``"FORTRAN_SUPPORT"`` (if Fortran support does not affect the rest) or a ``"VERSION"`` (if target development makes sure the code backward compatible ). Support for Features makes re-using the same target in more than one place much better, especially if the target is heavy to build (e.g. big external dependency) and have many different features that can be turned on/off that are consumed by different parts of our project.


Target definition file can describe more than one template/target
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

There is one-to-many relationship between target definition files and target definition. User can put multiple target/template names in the ``targets.cmake``. First parameter of ``generate_target()`` and ``declare_dependencies()`` is the name of the target/template, so the user code that handles target definition and dependencies can handle each target/template in different way. The feature was introduced to faciliate defining multiple similar targets in one file. It is used typically in external targets, where e.g. Boost defines multiple targets. 

Support for CMake code that do not produce targets
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Sometimes we need CMake to do something that does not necessary ends up being a new target, but rather modify existing target. This situation include dedicated code for definition of unit tests (of course unit tests can also be defined inside ``generate_target()`` that defines the executable), code that applies additional preprocesor definitions or old external targets that not use imported targets CMake mechanism. 


Comprehensive error checking
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

We spent a great deal of effort to catch as many usage errors as possible. In particular:

* We make sure that template modifiers and parameters share the same name space and report all violations. 
* System makes sure, that the actual template(s) you want to build actually produce targets.
* System makes sure that dependencies that do not generate targets are not dependent on templates that also do not produce targets.
* System detects lack of both ``generate_target()`` and ``apply_to_target()`` for internal projects.
* System detects and forbids variables that start with double underscore, as this can interfere with the Beetroot's inner working.
* System checks the types and possible values of all arguments against a declaration in ``targets.cmake``.

External dependencies are external to the Beetroot system
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
The system faciliates calling them, manages their external dependencies, build and install location, but ultimately it calls them as if they were a simple external CMake projects, builds them and installs them in the totally normal way.

Automatic superbuild by default
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

If there is at least one external target defined, all external dependencies are built in the superbuild phase, using the dependencies derrived from the template definitions. Only then the project build is called (implemented as a "external" dependency of all actual external dependencies), so all external dependencies are always already built and installed when you use them. If there are no external dependencies, CMake will build project directly.



List of the most visible features of the Beetroot
===========================================================

Targets are defined inside the function ``generate_target()`` in target definition file
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

In CMake targets (in contrast to variables) are internally represented by the object which lives in a global namespace, even if defined inside the function. Putting target definitions inside a function prevents leaking of temporary variables and pollution of the variable namespace. 

Dependencies of targets are set inside the function ``declare_dependencies()`` in target definition file
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Target dependencies are handled by function rather than data structure, which allows for maximum flexibility (dependencies can depend in complicated way on the target parameters/features). Because Beetroot structure needs dependencies to be resolved *before* target definition (and possibly be called multiple times on the same target), the only place to put them is in a dedicated user-supplied function. Code inside this function should be omnipotent, because it can be executed multiple times in a single run. The code will be executed only during the target declaration phase.


By default the code you write (targets.cmake) does not depend on your target name 
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Unless instructed otherwise, the system dictates the name you give to each target. This way targets' names are not fixed, and it is possible to have multiple instances of them. This fact is used to let the target definition files (``targets.cmake``) define whole family of targets parametrized by the target parameters and features. The beetroot guarantees, that for each distinct set of target parameter there will be a separate target defined and built.

There is only one type of user-supplied input file that defines the targets
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

All code that define targets and their dependencies can be placed inside so-called target definition files. These files can  be put anywhere in the project and must be named ``targets.cmake``, or be placed in the special subfolder ``cmake/targets`` and have an extension ``.cmake``. The latter files usually define external dependencies. The only thing that is influenced by the location of the file, is the value of the ``${CMAKE_CURRENT_SOURCE_DIR}`` CMake variable available in ``generate_target()`` user function.

The user file works by defining any of the following cmake variables: ``ENUM_TEMPLATES``, ``ENUM_TARGETS``, ``TARGET_PARAMETERS``, ``LINK_PARAMETERS``, ``TARGET_FEATURES`` ``TEMPLATE_OPTIONS`` and ``DEFINE_EXTERNAL_PROJECT`` and by defining any of the following functions: ``generate_targets()``, ``declare_dependencies()`` and ``apply_dependency_to_target()``. Of course, not all combinations of those definitions are legal and any violation of the legality of the definitions is cought and meaningfully reported to the user.

The other file a user needs to supply is a ``CMakeLists.txt``. This file serves as a point of entry. This file should consist of a standard boilerplate code, calls to the ``build_target()`` and finally a call to ``finalize()``. Standard CMake commands should not be used to define targets. The only purpose of this file is to specify what targets with what parameters must be build by calling a Beetroot function ``build_target()`` or ``get_target()`` and letting it do the work.


The role of the CMakeLists.txt is hugely downplayed
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

There is no need to use ``add_subdirectory()``, because Beetroot knows where to look for every managed target. That's why the only ``CMakeLists.txt`` that is needed is the one you manually call with ``cmake ..``. (Besides ``CMakeLists.txt`` inside CMake external dependencies of your project)

The location of the CMakeLists.txt is no longer relevant 
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

As long as the ``CMakeLists.txt`` is somewhere inside the root project and it adheres to the Beetroots' mandatory boilerplate code, its location is irrelevant. All components are searched for by name, not by folder, and the system requires them to be written in a way that all paths are absolute (it is achieved by simply prefixing filenames with the ``${CMAKE_CURRENT_SOURCE_DIR}``).

As long Beetroot is responsible for all targets in your code, you can simply copy a ``CMakeLists.txt`` from one subfolder of your project to another and they will build just fine there, resulting in exactly the same executable.


