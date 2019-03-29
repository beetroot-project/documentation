Getting started
===============


The Hello World
^^^^^^^^^^^^^^^

We will to start small, with the very simple C++ CMake build. 

Suppose this is our ``source.cpp``::

   // 'Hello World!' program 
   #include <iostream>
   #define STRINGIFY2(X) #X
   #define STRINGIFY(X) STRINGIFY2(X)
   #ifndef WHO
   #define WHO World
   #endif

   int main() {
      std::cout << "Hello " STRINGIFY(WHO) "!" << std::endl;
      return 0;
   }

For reference, to build it plain CMake, this would be our ``CMakeLists.txt``::

   cmake_minimum_required(VERSION 3.5)
   project(hello_simple)
   add_executable(hello_simple source.cpp)

Our file structure would like this::


| project_folder
| ├── build
| │   └── ...
| ├── source.cpp
| └── CMakeLists.txt
| 
| 

To be able to compile this code using Beetroot, we need first to make beetroot accessible to our build. If you use git, the preferred way to do that is to put the beetroot repository as a (shallow) submodule of your project, in folder ``cmake/beetroot``.

Then we put the following content in the ``targets.cmake``::

   set(ENUM_TEMPLATES HELLO_SIMPLE)
   
   function(generate_targets TEMPLATE_NAME)
      add_executable(${TARGET_NAME} "${CMAKE_CURRENT_SOURCE_DIR}/test.cpp")
   endfunction()

We also need to adjust the ``CMakeLists.txt`` to have the Beetroot actually executed::

   cmake_minimum_required(VERSION 3.13)
   include(../cmake/beetroot/beetroot_bootstrap)
   
   project(hello_simple)
   
   build_target(HELLO_SIMPLE)
   
   finalize()

Our final folder structure should look like this::


| project_folder
| ├── build
| │   └── ...
| ├── cmake
| │   ├── beetroot (beetroot clone)
| │   │   └── ...
| │   └── root.cmake
| ├── source.cpp
| ├── targets.cmake
| └── CMakeLists.txt
| 
| 


And finally we are set. Keep in mind, that Beetroot is designed to work best with middle and large size projects, so the amount of work to get the simplest C++ code compile is offset by the time we save when the project grows.

We compile it as usual::

   $ cd project_folder
   $ mkdir build
   $ cd build
   $ cmake .. && make 
   
       DECLARING  DEPENDENCIES  AND  DECIDING  WHETHER  TO  USE  SUPERBUILD
   
   No languages in project bootstrapped_hello_simple
   -- Discovering dependencies for HELLO_SIMPLE (HELLO_SIMPLE_f9fc6118c955867490b6f80bce90dc5b)...
   
   
   
       DEFINING  TARGETS  IN  PROJECT BUILD
       TESTS  disabled
   
   -- The CXX compiler identification is GNU 7.3.0
   -- Check for working CXX compiler: /home/adam/spack/opt/spack/linux-ubuntu16.04-x86_64/gcc-8.1.0/gcc-7.3.0-zclb4ttmy53mjkahiocmsqozhu6veriz/bin/g++
   -- Check for working CXX compiler: /home/adam/spack/opt/spack/linux-ubuntu16.04-x86_64/gcc-8.1.0/gcc-7.3.0-zclb4ttmy53mjkahiocmsqozhu6veriz/bin/g++ -- works
   -- Detecting CXX compiler ABI info
   -- Detecting CXX compiler ABI info - done
   -- Detecting CXX compile features
   -- Detecting CXX compile features - done
   -- Configuring done
   -- Generating done
   -- Build files have been written to: /home/adam/beetroot-examples/hello_simple/build
   
   Scanning dependencies of target bootstrapped_hello_simple
   [ 50%] Building CXX object CMakeFiles/bootstrapped_hello_simple.dir/source.cpp.o
   [100%] Linking CXX executable bootstrapped_hello_simple
   [100%] Built target bootstrapped_hello_simple
   $ ls
   hello_simple  CMakeCache.txt  CMakeFiles  cmake_install.cmake  Makefile
   $ ./hello_simple
   Hello World!

Now let's start complicating things. You may have noticed, that we have a macro parameter ``WHO`` in our C++ file, that can be used to change the program output. Let's do just that. After all, handling target parameters is one of the strongest sides of Beetroot. Let's modify our ``targets.cmake`` and insert definition of the parameter, which we will also call ``WHO``::

   set(ENUM_TEMPLATES HELLO_SIMPLE)
   
   set(TARGET_PARAMETERS 
      WHO SCALAR STRING "Beetroot"
   )
   
   function(generate_targets TEMPLATE_NAME)
      add_executable(${TARGET_NAME} "${CMAKE_CURRENT_SOURCE_DIR}/source.cpp")
      target_compile_definitions(${TARGET_NAME} PRIVATE "WHO=${WHO}")
   endfunction()

The name of the parameter does not need to match the name of the preprocessor macro. The formal syntax is this: ``TARGET_PARAMETERS`` is an array organized into 4-element tuples.

#. First element of the tuple is the name of the parameter, then
#. container type. There are three container types: ``OPTIONAL``, ``SCALAR`` and ``VECTOR``, and they correspond to the CMake options, scalars and lists.
#. Element type. At the moment the are 5 possible types: ``BOOL``, ``INTEGER``, ``PATH``, ``STRING`` and ``CHOICE(<colon-separated list of possible values>)``.
#. Default value. 

In the function body we need to tie the parameter with the target, and we do that in the usual CMake way, by using ``target_compile_definitions()``. All target parameters are always implicitely available in the function ``generate_targets``, so we can simply use them.

If we compile the program and run we get::

   $./hello_simple
   Hello Beetroot!

Let's say, that this file is our unit test and we need to compile three of them, one for the default string, and the other for a special string "Mars" and "Venus". It is easy with Beetroot, and by doing this we will demonstrate two ways of passing variables to targets. Let's re-write the ``CMakeLists.txt``::

   cmake_minimum_required(VERSION 3.13)
   include(../cmake/beetroot/beetroot_bootstrap)
   
   project(hello_simple)
   
   build_target(HELLO_SIMPLE)
   set(WHO "Venus")
   build_target(HELLO_SIMPLE)
   build_target(HELLO_SIMPLE WHO Mars)
   
   finalize()


After we build, we should get three executables: ``hello_simple1``, ``hello_simple2`` and ``hello_simple3``.::

   $./hello_simple1
   Hello Beetroot!
   $./hello_simple2
   Hello Venus!
   $./hello_simple3
   Hello Mars!

The ``targets.cmake`` defines a target _template_, that can be used to define as many targets, as there are unique combinations of target parameters. That is why the ``generate_targets()`` function requires user to use ``${TARGET_NAME}`` instead of hard-coded name, that is usual in standard CMake practice. The function will be called exactly once for each distinct ``${TARGET_NAME}`` that Beetroot found is required to satisfy the parameters.

Subprojects
^^^^^^^^^^^
Here you will learn how to combine targets together.

Suppose we have a program, that requires a function ``get_string`` from a library to run. The `hello_with_lib.cpp`::

	#include <iostream>
	#include "libhello.h"
	
	#ifndef LIBPAR
	#define LIBPAR 0
	#endif
	
	int main()
	{
	  int libpar = LIBPAR;
	  
	  std::cout << "Hello "<< get_string()<<"!"<< std::endl;
	  return 0;
	}

To compile it, we need a `libhello.h` that provides the ``get_string()``::

	#include<string>
	std::string get_string();

The library's implementation is in the file ``libhello.cpp``::

	#include "libhello.h"
	#define STRINGIFY2(X) #X
	#define STRINGIFY(X) STRINGIFY2(X)

	#ifndef WHO
	#define WHO World
	#endif

	std::string get_string() {
		return(STRINGIFY(WHO));
	}

The library depends on one macro: ``WHO`` that influences the text returned by the function.

We would like to have the ``hello_with_lib.cpp`` compiled and linked with the ``libhello``. Although there is nothing wrong with putting the additional CMake commands in the old ``targets.cmake`` file, it is not the best practice. Instead we will create two separate targets, so it will be easy to re-use the ``libhello`` by simply importing it.

Another thing to notice is that the Beetroot be default does not care about the location of the target definitions. Instead it scans recursively all the superproject files in search for files ``targets.cmake`` and subfolder structure ``cmake/targets/*.cmake``. Then it loads each fond file and learns the name of the targets/templates exposed there to build a mapping target/template name -> path of the target definition file, so user does not need to care about the paths anymore. On the other hand it requires that each each target/template name is unique across the whole superproject.

Let's create the following directory structure::


| superproject
| ├── cmake
| │   ├── beetroot (beetroot clone)
| │   │   └── ...
| │   └── root.cmake
| ├── hello_with_lib
| │   ├── hello_with_lib.cpp
| │   ├── CMakeLists.txt
| │   └── targets.cmake
| └── libhello
| │   ├── include
| │   │   └── libhello.h
| │   ├── source
| │   │   └── libhello.cpp
| │   └── targets.cmake
| └── CMakeLists.txt
| 
| 

This is the definition of the ``libhello/targets.cmake``::

   set(ENUM_TEMPLATES LIBHELLO)
   
   set(TARGET_PARAMETERS 
      WHO	SCALAR	STRING	"Jupiter"
   )
   
   function(generate_targets TEMPLATE_NAME)
      add_library(${TARGET_NAME} "${CMAKE_CURRENT_SOURCE_DIR}/source/libhello.cpp")
      add_source(${TARGET_NAME} PRIVATE "${CMAKE_CURRENT_SOURCE_DIR}/include/libhello.h") #For better IDE integration
      
      target_include_directories(${TARGET_NAME} PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include)
      target_compile_definitions(${TARGET_NAME} PRIVATE "WHO=${WHO}")
   endfunction()

Nothing new, except we use ``add_library`` instead of ``add_executable``. Adding ``libhello.h`` to sources is not strictly necessary, but is a good CMake practice, that helps various IDE generators generate better projects. 

This is the definition of the ``hello_with_lib/targets.cmake``::

   set(ENUM_TEMPLATES HELLO_WITH_LIB)
   
   function(declare_dependencies TEMPLATE_NAME)
      build_target(LIBHELLO WHO "Saturn")
   endfunction()
   
   function(generate_targets TEMPLATE_NAME)
      add_executable(${TARGET_NAME} "${CMAKE_CURRENT_SOURCE_DIR}/hello_with_lib.cpp")
   endfunction()

The new element, the ``declare_dependencies()`` function, is used to declare dependencies. It is a function, so user can build complex logic that turns certain dependencies on and off depending on the Target Parameters and Features. To declare a certain target/template a dependency we call a function ``build_target(<TEMPLATE_OR_TARGET_NAME> [<PARAMETERS>...])``. The API and behaviour is exactly the same, as in ``CMakeLists.txt``.

Let's step up our example and require that the ``HELLO_WITH_LIB`` is also parametrized by the parameter ``WHO``. There are two ways to do it. The most obvious one is simply to add the ``set(TARGET_PARAMETERS WHO SCALAR STRING "Jupiter")`` to the ``hello_with_lib/targets.cmake`` but that will not scale, if there are many parameters in question and they change. The better solution is to import the parameters using the special function designed for this purpose.

This is the definition of the ``hello_with_lib/targets.cmake``::

   set(ENUM_TEMPLATES HELLO_WITH_LIB)
   
   include_target_parameters_of(LIBHELLO)
   
   function(declare_dependencies TEMPLATE_NAME)
      build_target(LIBHELLO WHO "Saturn")
   endfunction()
   
   function(generate_targets TEMPLATE_NAME)
      add_executable(${TARGET_NAME} "${CMAKE_CURRENT_SOURCE_DIR}/hello_with_lib.cpp")
   endfunction()



Now let's complicate matters a bit and add two things. First imagine, that the ``hello_with_lib`` is also responsible for setting a macro variable in the client's code. Let's predend that this variable modifies behaviour of the header-only part of this library. Consequently will not change the library code. We only need to make sure, that clients linking to our library receive a new preprocessor macro::

   set(ENUM_TEMPLATES LIBHELLO)
   
   set(TARGET_PARAMETERS 
      WHO	SCALAR	STRING	"Jupiter"
   )
   
   set(LINK_PARAMETERS 
      LIBPAR	INTEGER	
   )
   
   function(generate_targets TEMPLATE_NAME)
      add_library(${TARGET_NAME} "${CMAKE_CURRENT_SOURCE_DIR}/source/libhello.cpp")
      add_source(${TARGET_NAME} PRIVATE "${CMAKE_CURRENT_SOURCE_DIR}/include/libhello.h") #For better IDE integration
      
      target_include_directories(${TARGET_NAME} PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include)
      target_compile_definitions(${TARGET_NAME} PRIVATE "WHO=${WHO}")
   endfunction()

   function(apply_dependency_to_target DEPENDEE_TARGET_NAME OUR_TARGET_NAME)
      target_compile_definitions(${DEPENDEE_TARGET_NAME} PRIVATE "LIBPAR=${LIBPAR}")
   endfunction()


