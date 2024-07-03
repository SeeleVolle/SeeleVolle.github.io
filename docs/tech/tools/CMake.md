# CMake 学习笔记

[参考官方Cmake Tutorial](https://cmake.org/cmake/help/latest/guide/tutorial)

### A Basic Starting Point

本章介绍了Cmake的基本使用，包括一系列基本命令:

+ `cmake_minimum_required(VERSION 3.10)`
+ `project(Tutorial VERSION 1.0)`
+ `set(CMAKE_CXX_STANDARD 11) set(CMAKE_CXX_STANDARD_REQUIRED True)`
+ `configure_file(TutorialConfig.h.in TutorialConfig.h)`
+ `add_executable(Tutorial tutorial.cxx)`
+ ```cmake
    target_include_directories(Tutorial PUBLIC
                          "${PROJECT_BINARY_DIR}"
                          "${PROJECT_SOURCE_DIR}/MathFunctions"
                          )
    ```

### Adding a Library

本章介绍了`CMakeLists.txt`如何添加一个库，以及添加编译选项来开启库的编译，主要命令包括：

+ `add_library(MathFunctions MathFunctions.cxx)`
+ `add_subdirectory(MathFunctions)`
+ `target_include_directories()`
+ `target_link_libraries(Tutorial PUBLIC MathFunctions)`
+ `PROJECT_SOURCE_DIR`
+ `if()`
+ `option(USE_MYMATH "Use tutorial provided math implementation" ON) `
+ `target_compile_definitions()`
+ ```cmake
    if(USE_MYMATH)
        target_compile_definitions(MathFunctions PRIVATE "USE_MYMATH")
        add_library(SqrtLibrary static mysqrt.cxx)
        target_link_libraries(MathFunctions PRIVATE SqrtLibrary)
    endif()
    ```

### Adding Usage Requirements for a Library

本章介绍了现代CMake方法，并给库添加个人的使用要求(控制库或可执行文件的连接和包含)，主要命令包括：

+ `CMAKE_CURRENT_SOURCE_DIR`
+ `target_compile_definitions()`
+ `target_compile_options()`
+ `target_include_directories()`
+ `target_link_directories()`
+ `target_link_options()`
+ `target_precompile_headers()`
+ `target_sources()`
+ `PUBLIC PRIVATE INTERFACE`: 三者之间有一定的区别。
    + `PUBLIC`: 当库A以PUBLIC方式链接到库B时，库A使用库B的接口，并且库A的接口也会使用库B的接口。这意味着链接到库A的目标（库或可执行文件）也会自动链接到库B，并且库B的接口属性（如头文件路径、宏定义等）也会传递给这些目标。
    + `PRIVATE`: 当库A以PRIVATE方式链接到库B时，库A使用库B的接口，但库A的接口不使用库B的接口。这意味着链接到库A的目标不会自动链接到库B，且库B的接口属性不会传递给这些目标。PRIVATE链接适用于库B仅在库A的内部实现中使用，而不通过库A的接口暴露给其他目标的情况。
    + `INTERFACE`: 当你设置一个目标的INTERFACE属性时，这些属性不会应用于该目标本身，而只会应用于链接了该目标的其他目标

### Adding Generator Expressions

本章介绍了生成表达式，生成表达式用于在编译期间生成信息，包括逻辑，信息和输出表达式，通常用于有条件的添加编译器编译选项：

+ `cmake-generator-expressions(7)`
+ `cmake_minimum_required()`
+ `set()`
+ `target_compile_options()`

### Installing and Testing

本章介绍了CMake支持的本地安装规则，一般需要指定`/lib`和`include`目录的安装, 以及CTest的测试框架和测试函数的编写。

编译完成后，在`build`目录下运行`cmake -VV`或者`cmake -C`可以进行CTest并看到测试结果

+ `install()`
+ `TARGETS FILES`
+ `DESTIANTION`
+ ```cmake
    if (TARGET SqrtLibrary)
        list(APPEND installable_libs SqrtLibrary)
    endif()
    ```
+ `enable_testing()`
+ `add_test()`
+ `function()`
+ `set_tests_properties()`
+ `ctest`
+ ```cmake
    function(do_test target arg result)
        add_test(NAME Test_${arg} COMMAND ${target} ${arg})
        set_tests_properties(Test_${arg} 
        PROPERTIES PASS_REGULAR_EXPRESSION ${result}
        )
    endfunction()
    ```

### Adding System Introspection

本章为`CMakeLists.txt`添加系统自检，根据目标平台是否具有某些特性来*添加代码*或*更改实现*

+ `CheckCXXSourceCompiles`
+ `target_compile_definitions()`
+ `check_cxx_compile_source()`
+ ```cpp
    #if defined(HAVE_LOG) and defined(HAVE_EXP)
    // TODO 5: If both HAVE_LOG and HAVE_EXP are defined,  use the following:
        double result = std::exp(std::log(x) * 0.5);
        std::cout << "Computing sqrt of " << x << " to be " << result
                << " using log and exp" << std::endl;
    // else, use the existing logic.
    #else
        double result = x;
    #endif
    ```

### Packaging an Installer

添加以下到CMakeLists.txt中，之后就可以调用`cpack`命令打包安装程序

```cmake
    include(InstallRequiredSystemLibraries)
    set(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/License.txt")
    set(CPACK_PACKAGE_VERSION_MAJOR "${Tutorial_VERSION_MAJOR}")
    set(CPACK_PACKAGE_VERSION_MINOR "${Tutorial_VERSION_MINOR}")
    set(CPACK_GENERATOR "TGZ")
    set(CPACK_SOURCE_GENERATOR "TGZ")
    include(CPack)
```

打包命令：
+ `cpack`
+ `cpack -G ZIP -C Debug`: 指定二进制生成器和多配置构建
+ `cpack --config CPackSourceConfig.cmake`: 创建完整源代码树的文档

### Selecting Static or Shared Libraries

+ `set_target_properties`
+ `option(BUILD_SHARED_LIBS "Build using shared libraries" ON)`
+ `message`
+ ```cmake
    if(CMAKE_BUILD_TYPE STREQUAL "Debug")
        # 设置Debug模式的特定编译器标志
        message(STATUS "Configuring for Debug mode")
        add_compile_definitions(MyExecutable PRIVATE DEBUG_MODE=1)
        target_compile_options(MyExecutable PRIVATE -g)
    elseif(CMAKE_BUILD_TYPE STREQUAL "Release")
    # 设置Release模式的特定编译器标志
        message(STATUS "Configuring for Release mode")
        add_compile_definitions(MyExecutable PRIVATE RELEASE_MODE=1)
        target_compile_options(MyExecutable PRIVATE -O3)
    endif()
    ```
