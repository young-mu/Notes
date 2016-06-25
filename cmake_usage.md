# cmake usage

## 选项
- `-H<SRC_DIR>` 指定源码目录
- `-B<BIN_DIR>` 指定输出文件目录
- `-D<VARIBALE>=<VALUE>` 给变量`VARIABLE`赋值`VALUE`
- `-G "Ninja | -GNinja"` 指定输出`Ninja`构建脚本
- `-G "Unix Makefiles"` 指定输出`Makefile`构建脚本（中间可不加空格）
- `-E chdir <DIR> <CMD>` 切换到`<DIR>`目录执行`<CMD>`命令

## 常用函数


### `CMAKE_MINIMUM_REQUIRED`

检查cmake最低版本

```c
CMAKE_MINIMUM_REQUIRED(VERSION 3.5)
```

### `PROJECT`

项目命名，可通过`PROJECT_NAME`变量访问

```c
PROJECT(test)
MESSAGE(STATUS "PROJECT_NAME = ${PROJECT_NAME}")
```

### `MESSAGE`

打印log，三种级别`STATUS`/`SEND_ERROR`/`FATAL_ERROR`

```c
MESSAGE(STATUS "report status")
MESSAGE(SEND_ERROR "report error and continue")
MESSAGE(FATAL_ERROR "report error and exit")
```

### `SET`

设置变量（变量名一般大写）

```c
SET(CMAKE_MODULE_PATH "{PROJECT_SOURCE_DIR}/submodule")

# 在函数或子项目CMakeLists.txt中若加PARENT_SCOPE， 则退出函数或回到Root CMakeListst.txt后当前设置作用于该变量
SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall" PARENT_SCOPE)
```

### `INCLUDE`

引入子Cmake子模块

```c
# 设定源码根目录的submodule目录为module目录，并引入该目录下的test.cmake
SET(CMAKE_MODULE_PATH "{PROJECT_SOURCE_DIR}/submodule")
INCLUDE(test)
```

### `LIST`

对列表进行操作

```c
LIST(LENGTH ARGV argv_len) # 将ARGV列表的长度赋值给argv_len
LIST(REVERSE ARGV) # 反转ARGV列表本身
LIST(SORT ARGV) # 将ARGV列表本身按照字母顺序排列
LIST(REMOVE_DUPLICATES ARGV) # 将ARGV中重复项删掉
```

### `STRING`

对字符串进行操作

```c
# 注意，列表间隔符;不能处理，字符串中没有，系统自动加的
SET(MY_LIST "1-" "2-" "3-")
MESSAGE(STATUS "MY_LIST = ${MY_LIST}") # 1-;2-;3-
STRING(REPLACE "-" " " _MY_LIST ${MY_LIST})
MESSAGE(STATUS "_MY_LIST = ${_MY_LIST}") # 1 2 3
```

### `FILE`

对文件进行操作

```c
# 查找当前src目录下所有C文件，加到SRC_LIST列表中，用绝对路径表示
FILE(GLOB SRC_LIST src/*.c)
```

### `ADD_DEFINITIONS`

添加宏定义参数

```c
ADD_DEFINITIONS(-DDEBUG) # 加双引号也可以
```

### `ADD_CUSTOM_TARGET`

加入自定义目标

```c
# 执行make custom_target时，选择源码根目录为工作目录，执行pwd命令
ADD_CUSTOM_TARGET(custom_target
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
    COMMAND pwd
)

# 执行make custom_target时，切换目录，执行pwd命令，最好加VERBATIM保护命令执行
ADD_CUSTOM_TARGET(custom_target
    COMMAND ${CMAKE_COMMAND} -E chdir ${CMAKE_BINARY_DIR} pwd
    COMMENT "Haha, I'm custom_target!!!"
    VERBATIM
)
```

### `ADD_SUBDIRECTORY`

增加子目录，引入子目录的CMakeLists.txt

```c
# 将在PROJECT_BINARY_DIR下生成gpio目录
ADD_SUBDIRECTORY("${PROJECT_SOURCE_DIR}/gpio")
```

### `ADD_EXECUTABLE`

编译可执行程序

```c
SET(SRC_LIST test.c)
ADD_EXECUTABLE(test ${SRC_LIST})
```

### `ADD_LIBRARY`

编译库（静态和动态）

```c
# 编译静态库libtest.a
PROJECT(test)
ADD_LIBRARY(${PROJECT_NAME} STATIC ${SRC_LIST})

# 编译动态库libtest.so（linux平台）和libtest.dylib（Mac平台）
PROJECT(test)
ADD_LIBRARY(${PROJECT_NAME} SHARED ${SRC_LIST})

# 引入静态库
ADD_LIBRARY(test_util STATIC IMPORTED)
SET_PROPERTY(TARGET test_util PROPERTY IMPORTED_LOCATION ${TEST_LIB_PATH})
TARGET_LINK_LIBRARIES(${PROJECT_NAME} test_util)
```

### `SET_TARGET_PROPERTIES`

更改目标属性

- `PREFIX` 默认为lib
- `SUFFIX` 默认为so（Linux平台）和dylib（Mac平台）
- `OUTPUT_NAME` 默认为Target名字，即${PROJECT_NAME}

```c
SET_TARGET_PROPERTIES(${PROJECT_NAME} PROPERTIES PREFIX "")
SET_TARGET_PROPERTIES(${PROJECT_NAME} PROPERTIES SUFFIX ".so")
SET_TARGET_PROPERTIES(${PROJECT_NAME} PROPERTIES OUTPUT_NAME "foo")
```

### `TARGET_LINK_LIBRARIES`

将库文件链接到生成的目标

```c
TARGET_LINK_LIBRARIES($PROJECT_NAME} libfoo.so}
```

### `INCLUDE_DIRECTORIES`

增加头文件搜素目录

```c
INCLUDE_DIRECTORIES(${PROJECT_SOURCE_DIR}/include)
```

### `LINK_DIRECTORIES`

增加库搜索目录

```c
LINK_DIRECTORIES(${PROJECT_SOURCE_DIR}/lib)
```

### `INSTALL`

安装文件或目录

```c
# 安装${CMAKE_BINARY_DIR}/bin/hello到${CMAKE_INSTALL_PREFIX}/ABC目录中（自动创建）
INSTALL(FILES ${CMAKE_BINARY_DIR}/bin/hello DESTINATION ABC)

# 安装${PROJECT_SOURCE_DIR}/test目录到${CMAKE_INSTALL_PREFIX}/ABC目录中
INSTALL(DIRECTORY test DESTINATION ABC)

# 安装${PROJECT_NAME}下的库文件到${CMAKE_INSTALL_PREFIX}/ABC目录中
INSTALL(TARGETS ${PROJECT_NAME} LIBRARY DESTINATION ABC)
```

### `EXECUTE_PROCESS`

执行子程序，子程序内不能执行`cd`命令，若更改工作目录，必须重新执行一次该命令

```c
SET(build_dir my_build)
EXECUTE_PROCESS(
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/lib/mbedtls
    COMMAND cmake -H. -B${build_dir}
    OUTPUT_QUIET
)
EXECUTE_PROCESS(
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/lib/mbedtls/${build_dir}
    COMMAND make
    OUTPUT_QUIET
    ERROR_QUIET
)
```

### `FIND_PACKAGE`

查找是否有指定的软件

```c
# 检查环境变量中是否有python命令
FIND_PACKAGE(PYTHONINTERP)
IF(NOT PYTHONINTERP_FOUND)
	MESSAGE(STATUS "Python is NOT found!")
ELSE()
	MESSAGE(STATUS "PYTHON_EXECUTABLE = ${PYTHON_EXECUTABLE}")
ENDIF()
```

## 常用变量

- `CMAKE_COMMAND` 默认cmake
- `CMAKE_VERSION` 3.5.2
- `CMAKE_C_COMPILER` 默认cc（设置该变量必须指定绝对目录且一定要在`PROJECT()`前面）
- `CMAKE_CXX_COMPILER` 默认c++
- `CMAKE_C_FLAGS` 编译选项（默认空），可在cmake命令时直接append
- `CMAKE_C_FLAGS_RELEASE` RELEASE版本编译选项（默认-O3 -DNDEBUG）
- `CMAKE_C_FLAGS_DEBUG` DEBUG版本编译选项（默认-g）
- `CMAKE_SHARED_LINKER_FLAGS` 链接动态库的选项（默认空）
- `CMAKE_BUILD_TYPE` 默认空
- `CMAKE_SYSTEM` Mac平台Darwin-15.5.0
- `CMAKE_SYSTEM_NAME` Mac平台Darwin
- `CMAKE_SYSTEM_VERSION` Mac平台15.5.0
- `CMAKE_SYSTEM_PROCESSOR` x86_64
- `PROJECT_SOURCE_DIR` 当前目录绝对路径（如.或subdir，必须声明project）
- `PROJECT_BINARY_DIR` 二进制文件输出目录绝对路径（如./build或./build/subdir）
- `CMAKE_CURRENT_SOURCE_DIR` 当前cmake执行所在的目录
- `CMAKE_SOURCE_DIR` 为-H指定的目录绝对路径（如./）
- `CMAKE_BINARY_DIR` 为-B指定的目录绝对路径（如./build）
- `PROJECT_NAME` 项目名称
- `EXECUTABLT_OUTPUT_PATH` 可执行程序的输出目录（默认空）
- `LIBRARY_OUTPUT_PATH` 库的输出目录（默认空）
- `CMAKE_VERBOSE_MAKEFILE` 输出详细的编译链接信息（默认FALSE）
- `ENV{HOME}` 获取SHELL环境变量（如HOME）
- `CMAKE_INSTALL_PREFIX` 安装文件或目录的输出目录（默认为`/usr/local`）
- `CMAKE_MODULE_PATH` Cmake模块路径
- `CMAKE_GENERATOR` 为-G指定的构建后端（`Unix Makefiles`或`Ninja`）
- `CMAKE_MAKE_PROGRAM` 为-G指定的构建工具（`make`或`ninja`绝对路径）

## 函数

- `ARGC` 输入参数个数
- `ARGN` 超过期望的输出参数（分号分隔）
- `ARGV` 所有输入参数（分号分隔）
- `ARGVn` 代表第n个参数

```c
# 定义函数foo，只有一个期望的参数
function(foo arg)
	MESSAGE(STATUS "arg = ${arg}")
    MESSAGE(STATUS "ARGC = ${ARGC}")
    MESSAGE(STATUS "ARGN = ${ARGN}")
    MESSAGE(STATUS "ARGV = ${ARGV}")
    MESSAGE(STATUS "ARGV0 = ${ARGV0}")
    RETURN()
    MESSAGE(STATUS "ARGV1 = ${ARGV1}")
endfunction()

foo(1 2 3)

-- arg = 1
-- ARGC = 3
-- ARGN = 2;3
-- ARGV = 1;2;3
-- ARGV0 = 1
```

## 判断

### 判断平台

- `IF(CMAKE_SYSTEM_CMAKE MATCHES "Linux")` # Linux平台
- `IF(APPLE)` # Mac平台
- `IF(WIN32)` # Windows平台

### 判断环境变量是否为空

Linux和Mac平台都支持

```c
IF(NOT "$ENV{MY_CMAKE_CC}" STREQUAL "")
    SET(CMAKE_C_COMPILER $ENV{MY_CMAKE_CC})
ENDIF()
PROJECT(project)
```

## 循环

```c
FOREACH(VAR ${LINT_SCOPES})
    MESSAGE(STATUS "VAR = ${VAR}")
    SET(_LINT_SCOPES "${_LINT_SCOPES} ${VAR}")
ENDFOREACH()
```

## 语法注意事项

1. 函数名和关键字（如`IF`）大小写均可，但**系统变量必须大写**
2. 函数名和关键字与左括号有无空格均可
3. 左扩后右侧和右括号左侧有无空格均可

## 参考
1. [CMake Wiki](https://cmake.org/Wiki/CMake)
2. [CMake Useful Variables](https://cmake.org/Wiki/CMake_Useful_Variables)
