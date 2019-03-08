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


