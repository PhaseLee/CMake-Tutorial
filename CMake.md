# 概述

&emsp;&emsp;`CMake`是一个跨平台`C/C++`编译工具，使得同一个`C/C++`项目可在不同的`IDE`上编译运行，`CMake`中所有的操作都可以通过`CMakeLists.txt`文件完成。`CMake`的基本使用流程：

```xxx
1.编写CMakeLists.txt
2.使用CMake执行CMakeLists文件生成跨平台的Makefile文件
3.使用Make执行Makefile文件编译工程形成可执行文件
```

&emsp;&emsp;`Make`使用`Makefile`指导编译器进行源文件的批量编译，对于大型项目而言编写`Makefile`仍然是很复杂的工作，并且编写的`Makefile`跨平台性较差，因此可使用`CMake`执行`CMakeLists`生成跨平台的`Makefile`，再通过`Make`执行`Makefile`完成编译。

```xxx
CMakeLists ----> CMake --生成--> Makefile ----> Make --编译--> 可执行文件
```

# `GCC`编译器介绍

&emsp;&emsp;`GCC(GNU C Compiler)`是一个跨平台`C`交叉编译器(交叉编译是指在一个架构的平台编译出另一个架构的平台可运行的机器码)。`GCC`命令的一般格式：

```shell
gcc [option] source-file [option] target-file
```
## 编译步骤

&emsp;&emsp;编译命令示例：

```shell
# 将指定源文件编译为可执行文件，-o可指定编译生成的文件名及存放目录
# -Wall指定打印所有警告信息
# 注意编译时无需指定头文件，GCC编译器会根据源文件中的#include指令自动搜索头文件
gcc -Wall ./main.c ./test1.c ./test2.c -o ./build/main.o
```

&emsp;&emsp;`GCC`编译器的编译步骤：

```xxx
1.预处理器CPP - 预编译(预编译源文件生成.i文件，进行一些文本替换工作) : gcc -E main.c -o main.i
2.编译器GCC   - 编译生成汇编代码(编译.i文件生成.s文件) : gcc -S main.i -o main.s
3.汇编器AS    - 编译生成机器代码(编译.s文件生成目标文件) : gcc -c main.s -o main.o
4.链接器LD    - 链接目标文件生成可执行文件 : gcc main.o -o main
```

### 预处理

&emsp;&emsp;**预处理时会将头文件完整的拷贝到源文件的`#include`处，因此后续编译阶段编译生成的目标文件实际已经包含了头文件内容**。

```shell
# 预处理步骤进行一些文本拷贝或替换工作，预处理器CPP会处理源文件中以"#"开头的文本
gcc -E ./main.c -o ./build/main.i
```

### 编译

```shell
# -c命令为只编译不链接
# -c命令可用于源文件，也可用于.i文件
gcc -c ./main.c -o ./build/main.o
gcc -c ./build/main.i -o ./build/main.o
```

### 链接

```shell
# 链接时可以不指定输出文件的后缀名
gcc ./main.o ./test1.o ./test2.o -o ./build/main
```

### 头文件处理

&emsp;&emsp;`C`程序中的头文件包含有两种形式：

```c
// 使用 #include <...> 包含头文件时GCC编译器会首先在系统头文件目录下寻找指定文件 
#include <stdio.h>
// 使用 #include "..." 一般包含系统外的头文件
#include "main.h"
```

&emsp;&emsp;头文件与源文件不在同意路径时需使用命令选项显式指定头文件搜索路径：

```shell
# -I<include-directory>指定头文件搜索路径
# 对于C标准头文件，GCC会首先在系统头文件目录下寻找，所以不必显式指定
gcc -Wall -I./inc/ ./src/main.c ./src/test1.c ./src/test2.c -o ./build/main
```

### 链接库

&emsp;&emsp;可将多个目标文件打包为链接库提供给别的项目使用：

```shell
# 编译为目标文件
gcc -c -I./inc/ ./src/test1.c -o ./build/test1.o
gcc -c -I./inc/ ./src/test2.c -o ./build/test2.o

# 打包目标文件为库文件
# ar cr <lib-name> <target1 target2 ... >
ar cr ./lib/libtest.a ./build/test1.o ./build/test2.o
# ar t命令可查看链接库由哪些目标组成
ar t ./lib/libtest.a

# 链接库文件
# 链接库文件与目标文件
# 与编译阶段已经将头文件拷贝到了源文件中，即目标文件中已经包含头文件内容，无需再使用-I命令
# 链接库文件与目标文件时需要注意链接顺序，调用库文件的目标文件应在库文件之前
gcc ./build/main.o ./lib/libtest.a -o ./build/main
# 链接库文件与源文件
# 链接库文件与源文件时需要注意链接顺序，调用库文件的源文件应在库文件之前
gcc -I./lib ./src/main.c ./lib/libtest.a -o ./build/main
# 如果使用了-L指定库文件搜索路径且库文件名符合lib<name>的形式，可直接使用-l<name>命令代替库文件路径
# 如以上的使用方式如标准库中的-lm(标准math库)
gcc -I./lib -L./lib ./src/main.c -ltest.a -o ./build/main
```

&emsp;&emsp;链接库类型：

```xxx
STATIC(静态链接库):
  链接静态库时会将静态库的机器码拷贝到可执行文件中，这种方式可能会造成可执行文件过大，如果有多个程序调用同一个静态库将导致内存浪费，格式为.a(windows是.lib)
SHARED(动态链接库):
  链接动态库时只会将动态库的地址信息拷贝到可执行文件中，运行时先将动态库加载到内存，可执行文件直接在内存中查找动态库，实现轻量的可执行文件及库的共享性，动态库还可实现更新库文件时不用重新编译整个工程，格式为.so(windows是.dll)
```

## 常用`GCC`选项

### 编译选项

```xxx
-c                只编译不链接
-o                指定目标名称(缺省时编译生成的目标名称为a.out)，例：gcc -o main.exe main.c
-S                只激活预处理和编译，即将文件编译为汇编代码，例：gcc -S main.c
-E                预处理
-D<MACRO>         以字符串1定义<MACRO>宏
-D<MACRO>=<DEF>   以指定字符串<DEF>定义<MACRO>宏
-U<MACRO>         取消对<MACRO>的宏定义
-g                生成调试信息
-I<DIRECTORY>     指定额外的头文件搜索路径<DIRECTORY>
-O0               不进行优化处理
-O(或-O1)         优化生成代码
-O2               进一步优化
-O3               比-O2更进一步优化
-Od               禁用优化(默认值)
-w                不生成任何警告信息
-Wall             生成所有警告信息
-C                在预处理时不删除注释信息
-M                生成文件关联的信息，包含目标文件所依赖的所有源代码，例如 gcc -M main.c
-MM               与-M相同，但将忽略由#include造成的依赖关系
-MD               与-M相同，但输出将导入到.d文件里
-MMD              与-MM相同，但输出将导入到.d文件里
```

### 链接选项

&emsp;&emsp;参考[`GCC官方手册-链接选项`](https://gcc.gnu.org/onlinedocs/gcc-11.2.0/gcc/Link-Options.html#Link-Options)

```xxx
-L<DIRECTORY>       指定额外的库函数搜索路径<DIRECTORY>
-l<LIBRARY>         链接时搜索指定的函数库<LIBRARY>
-T<script>          指定链接脚本
-WL,<opt1,opt2,...> 将指定的选项传递给链接器
```


# `CMakeLists`语法

&emsp;&emsp;`CMakeLists.txt`中的命令不区分大小写，但变量名区分大小写，`CMakeLists`包含一些特殊功能的内置变量，也可以自定义变量。`CMakeLists.txt`中可定义使用函数。

## 项目基本信息

### `CMake`版本

```cmake
# 指定项目要求CMake的最低要求，此命令应当置于最先
cmake_minimum_required(VERSION 3.12)
```

### `project`

&emsp;&emsp;`project`命令用于指定工程名、基本工程信息等：

```cmake
# 此命令隐式定义了两个变量：[project-name]_BINARY_DIR 和 [project-name]_SOURCE_DIR
# 工程名变化将导致隐士指定的变量名跟随变化，CMake预定义了两个变量用以代替：
# PROJECT_BINARY_DIR 和 PROJECT_SOURCE_DIR 
# PROJECT_SOURCE_DIR即工程级CMakeLists.txt所在的目录
project(HelloWorld)
```

&emsp;&emsp;`project`命令可以同时指定其它基本工程信息：

```cmake
# 指定工程支持的语言，值可以使用 "" 包括，也可以省略 "" ，
# 当存在"DESCRIPTION","VERSION"等关键字时语言名之前必须添加LANGUAGES关键字
# CMake中参数之间使用空格隔开
project(HelloWorld LANGUAGES "C CXX JAVA")

# 指定工程描述信息，将为变量 PROJECT_DESCRIPTION 赋值
project(HelloWorld DESCRIPTION "test for cmake")

# 指定工程版本号，将为变量 PROJECT_VERSION 赋值
project(HelloWorld VERSION 1.0.0.0)
```

## 变量

&emsp;&emsp;`CMake`中有隐式定义的变量和显式定义的变量，`project`命令隐式定义的`PROJECT_DESCRIPTION`等变量即为隐式定义的变量，使用`set`命令定义的变量即为显式定义的变量。

### `set`

&emsp;&emsp;`set`命令用于在当前作用域下定义或设置变量，如果当前作用域下不存在指定的变量名，定义新变量并赋值，如果存在则直接赋值。变量的新值仅对当前作用域有效，离开当前作用域，如果域外存在同名变量，将使用域外同名变量的值。

```cmake
# 普通变量
# 设置当前作用域中名为"variable"的变量值为value,使用 ${variable} 使用新值
set(variable value)
# 设置当前作用域中名为"variable"的变量值为value1;value2;value3(使用";"连接)
set(variable value1 value2 value3)
# 设置0个值时将变量变为未设置状态，相当于unset命令
set(variable)
# 存在 "PARENT_SCOPE" 关键字，则设置上层作用域中名为"variable"的变量值为value,当前作用域不变
set(variable value PARENT_SCOPE)

# 环境变量
# 设置环境变量variable的值为value,使用 $ENV{variable} 使用新值
set(ENV{variable} value)

# 缓存变量(即全局变量，同一个CMake工程下的所有位置都可以使用)
# CACHE是缓存变量的标志，type指定变量类型，docstring为简要说明，必须为字符串
# 如果带上FORCE关键字，则表明强制修改缓存变量的值
set(<variable> <value...> CACHE <type> <docstring> [FORCE])
```

### `unset`

&emsp;&emsp;`unset`命令用于清空作用域下的变量：

```cmake
# 普通变量
# 清空当前作用域下的变量variable,清空后当前作用域调用该变量将产生警告
unset(variable)
# 清空上层作用域下的变量variable
unset(variable PARENT_SCOPE)

# 环境变量
unset(ENV{variable})

# 缓存变量
unset(variable CACHE)
```

## 函数和宏

### 函数

```cmake
# 无参函数
function(func1)
  set(string "Hello world.")
  message("func1 : ${string}")
  # return用于从函数退出，并非返回一个值
  return()
endfunction()
# 调用func1()
func1()

# 有参函数
function(func2 arg1 arg2 arg3)
  set(string "${arg1}-${arg2}-${arg3}")
  message("func2 : ${string}")
  # return用于从函数退出，并非返回一个值
  return()
endfunction()
# 调用func2()
func2("2022" "03" "06")
```

### 宏

```cmake
# 宏的定义及使用与函数一致，宏与函数的区别在于，函数会创建新的作用域，而调用宏只是代码块替换
macro(macro1 arg1 arg2)
  set(string "${arg1}-${arg2}")
  message("macro1 : ${string}")
endmacro()
macro1("aa" "bb")
```

## 流程控制

### 条件判断

```cmake
# 逻辑运算：NOT AND OR
# 一元操作符：EXIST,COMMAND,DEFINED
# 二元操作符：EQUAL,MATCHES,LESS,GREATER,LESS_EQUAL,GREATER_EQUAL
# 布尔值 true  : 1,ON,YES,TRUE,Y,非0的值
# 布尔值 false : 0,OFF,NO,N,FALSE,空字符串,以-NORFOUND结尾的字符串

function(equals a b)
  # 注意必须有空格
  if(${a} MATCHES ${b})
    message("${a} equals ${b}")
  elseif(NOT(${a} MATCHES ${b}))
    message("${a} not equals ${b}")
  endif()
endfunction()
equals(10 10)
equals(10 11)

function(compare a b)
  if(a GREATER b)
    message("${a} > ${b}")
  elseif(a LESS b)
    message("${a} < ${b}")
  else()
    message("${a} = ${b}")
  endif()
endfunction()
compare(11 10)
compare(11 12)
compare(12 12)
```

### 循环

```cmake
# 循环控制语句：break(),continue(),return()

# while
set(i 10)
while(${i} GREATER_EQUAL 0)
  message("this time i = ${i}")
  # 数学运算：math(EXPR <variable> <"expression">)
  math(EXPR i "${i} - 1")
endwhile()

# foreach
# 1.foreach(<loop_var> <items>),用于循环将items中的值赋给loop_var
foreach(i 1 2 3 4 5)
  message("this time i = ${i}")
endforeach()
# 2.foreach(<loop_var> RANGE <start> <stop> [step]),从start以step的步进增加到stop
foreach(i RANGE 1 5 1)
  message("this time i = ${i}")
endforeach()
```

### 模块

```cmake
TODO
```

## `CMake`项目构建

&emsp;&emsp;现代编译器的主要工作流程为：

```xxx
编译 :
|---源文件1 -> 编译器 -> 目标文件1
|---源文件2 -> 编译器 -> 目标文件2
...
|---源文件n -> 编译器 -> 目标文件n

链接 : 
|---链接库 + 目标文件1 + 目标文件2 + ... + 目标文件n -> 链接器 -> 可执行文件
```
&emsp;&emsp;使用`CMakeLists.txt`构建项目的一般步骤为：

```xxx
1.寻找源文件
2.使用源文件构建目标(add_executable()或add_library())
3.添加头文件搜索路径
4.链接目标(target_link_libraries())
```

### 常用命令

```cmake
# 使用指定的源文件生成一个可执行文件并添加到工程中
# name指定可执行文件名，必须在工程中全局唯一，且全局可用，可执行文件的具体后缀取决于具体平台(可设置)
add_executable(<name> [WIN32] [MACOSX_BUNDLE] [EXCLUDE_FROM_ALL] source1 [source2...])

# 使用指定的源文件生成一个库并添加到工程中
# name指定库文件名，必须在工程中全局唯一，且全局可用，库文件的具体后缀取决于具体平台(可设置)
add_library(<name> [STATIC|SHARED|MODULE] [EXCLUDE_FROM_ALL] source1 [source2...])

# 添加一个子目录到构建中
# source_dir指定CMakeLists.txt及源文件所在的目录
# binary_dir指定该子目录的构建输出目录
add_subdirectory(source_dir [binary_dir] [EXCLUDE_FROM_ALL])

# 收集指定目录dir中所有源文件的名称并以列表形式存储在指定的变量variable中
aux_source_directory(<dir> <variable>)

# 添加头文件搜索路径，当前CMakeList.txt中的所有目标，以及所有在其调用点之后添加的
# 子目录中的所有目标将具有此头文件搜索路径
include_directories([AFTER|BEFORE] [SYSTEM] dir1 [dir2...])

# 向指定目标添加头文件搜索路径，target必须提前由add_executable()或add_library()定义
target_include_directories(<target> [SYSTEM] [AFTER|BEFORE] <INTERFACE|PUBLIC|PRIVATE> 
                           [items1...] [<INTERFACE|PUBLIC|PRIVATE> [items2...] ...])

# 将目标链接到给定的库，target必须由add_executable()或add_library()定义
# 示例：target_link_libraries(target1 target2 target3 target4 target5)
target_link_libraries(<target> [item1 [item2 [...]]] [[debug|optimized|general] <item>]...)

# 设置目标属性
# 示例：set_target_properties(target1 target2 target3 PROPERTIES PREFIX "" SUFFIX ".lib")
set_target_properties(target1 target2 ... PROPERTIES prop1 value1 prop2 value2 ...)

# 为指定的目标添加宏定义，添加的宏定义将影响源码中的预编译设置
target_compile_definitions(<target>
                           <INTERFACE|PUBLIC|PRIVATE> [items1...]
                           [<INTERFACE|PUBLIC|PRIVATE> [items2...] ...])

# 为目标添加编译选项/标志
target_compile_options(<target> [BEFORE] 
    <INTERFACE|PUBLIC|PRIVATE> [item1...] [<INTERFACE|PUBLIC|PRIVATE> [item1...] ...])
# 为目标添加链接选项/标志
# PRIVATE : 应用于给定目标，不传递给与给定目标相关的目标
# INTERFACE : 应用于给定目标，传递给与给定目标相关的目标
# PUBLIC : 应用于给定目标和使用它的目标
target_link_options(<target> [BEFORE] 
    <INTERFACE|PUBLIC|PRIVATE> [item1...] [<INTERFACE|PUBLIC|PRIVATE> [item1...] ...])

# 编译/链接当前目录及以下目录时使用
add_compile_options(<option>...)
add_link_options(<option>...)
```

### 基本项目构建

&emsp;&emsp;可以仅使用工程根目录的`CMakeLists.txt`构建整个项目，工程结构：

```xxx
Project
|---build(存放编译文件)
|---out
    |---lib(存放链接库)
    |---bin(存放可执行文件)
|---CMakeLists.txt(工程构建目录下必须存在一个CMakeLists.txt，可用于整个工程的配置)
|---src
    |---main.h
    |---main.c
    |---test
        |---test1
            |---test1.h
            |---test1.c
        |---test2
            |---test2.h
            |---test2.c
            |---test3
                |---test3.h
                |---test3.c
```

`main.c`

```c
#include <stdio.h>
#include "main.h"
#include "test1.h"
#include "test2.h"

int main() {
    printf("main.\r\n");
    test1();
    test2();
    return 0;
}
```

`test1.c`

```c
#include <stdio.h>
#include "test1.h"

void test1() {
    printf("test1.\r\n");
}
```

`test2.c`

```c
#include <stdio.h>
#include "test2.h"
#include "test3.h"

void test2() {
    printf("test2.\r\n");
    test3();
}
```

`test3.c`

```c
#include <stdio.h>
#include "test3.h"

void test3() {
    printf("test3.\r\n");
}
```

`CMakeLists.txt`

```cmake
# 工程基本配置
cmake_minimum_required(VERSION 3.16)
project(HelloWorld LANGUAGES C)
set(CMAKE_C_STANDARD 99)

# 设置编译目标输出路径
set(EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/out/bin)
set(LIBRARY_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/out/lib)

# 将指定路径下的所有源文件路径存放在指定变量中
aux_source_directory(${PROJECT_SOURCE_DIR}/src/test/test1 source_test1)
# 使用指定源文件构建目标
add_library(library_test1 ${source_test1})
# 为目标指定头文件搜索路径，指定为PUBLIC的头文件搜索路径也可被链接此目标的目标使用
target_include_directories(library_test1 PUBLIC  ${PROJECT_SOURCE_DIR}/src/test/test1)

# 将指定路径下的所有源文件路径存放在指定变量中  
aux_source_directory(${PROJECT_SOURCE_DIR}/src/test/test2 source_test2)
aux_source_directory(${PROJECT_SOURCE_DIR}/src/test/test2/test3 source_test3)
# 使用指定源文件构建目标
add_library(library_test2 ${source_test2} ${source_test3})
# 为目标指定头文件搜索路径，指定为PUBLIC的头文件搜索路径也可被链接此目标的目标使用
target_include_directories(library_test2 
    PUBLIC  ${PROJECT_SOURCE_DIR}/src/test/test2
    PRIVATE ${PROJECT_SOURCE_DIR}/src/test/test2/test3)

# 显式设置编译输出目标文件的前后缀
set_target_properties(library_test1 library_test2 PROPERTIES PREFIX "" SUFFIX ".lib")

# 将指定路径下的所有源文件路径存放在指定变量中
aux_source_directory(${PROJECT_SOURCE_DIR}/src source_src)
# 使用指定源文件构建目标
add_executable(HelloWorld ${source_src})
# 为目标指定头文件搜索路径
target_include_directories(HelloWorld PUBLIC ${PROJECT_SOURCE_DIR}/src)
# 显式设置编译输出目标文件的前后缀
set_target_properties(HelloWorld PROPERTIES PREFIX "" SUFFIX ".bin")

# 链接指定目标
target_link_libraries(HelloWorld library_test1 library_test2)
```
&emsp;&emsp;使用命令行构建工程：

```shell
# 提前在项目的根目录下创建build文件夹
cd /HelloWorld/build
# 在build文件夹中运行cmake命令，构建生成的Makefile将存放在此文件夹中而不会污染工程
# cmake运行的路径中必须包含CMakeLists.txt文件
sudo cmake ../
# 使用make命令执行biuld中的Makefile完成编译
sudo make ./
# 运行bin下生成的可执行文件
sudo ../out/bin/HelloWorld.run
# 清理Makefile
sudo rm -r ../build/*
```


### 多模块项目构建

&emsp;&emsp;可以为项目中的每个模块目录中都添加一个`CMakeLists.txt`构成多模块项目，工程结构：

```xxx
Project
|---build(存放编译文件)
|---out
    |---lib(存放链接库)
    |---bin(存放可执行文件)
|---CMakeLists.txt(工程构建目录下必须存在一个CMakeLists.txt，可用于整个工程的配置)
|---src
    |---CMakeLists.txt
    |---main.h
    |---main.c
    |---module1
        |---CMakeLists.txt
        |---module1.h
        |---module1.c
    |---module2
        |---CMakeLists.txt
        |---module2.h
        |---module2.c
```

&emsp;&emsp;`main.c`的内容：

```c
#include <stdio.h>
#include "main.h"
#include "module1.h"
int main() {
    printf("I am main().\r\n");
    test1();
    return 0;
}
```

&emsp;&emsp;`module1.c`的内容：

```c
#include <stdio.h>
#include "module1.h"
#include "module2.h"
void test1() {
    printf("I am test1().\r\n");
    test2();
}
```

&emsp;&emsp;`module2.c`的内容：

```c
#include <stdio.h>
#include "module2.h"
void test2() {
    printf("I am test2().\r\n");
}
```

&emsp;&emsp;工程级`CMakeLists.txt`(`Project/CMakeLists.txt`)内容：

```cmake
# 指定cmake版本要求
cmake_minimum_required(VERSION 3.16)
# 指定工程名，工程信息等
project(CMakeDemo LANGUAGES C DESCRIPTION "Test demo for CMake" VERSION 1.0.0)
# 指定C标准
set(CMAKE_C_STANDARD 99)
# 添加构建子目录
add_subdirectory(./src)
```

&emsp;&emsp;`src`下`CMakeLists.txt`内容：

```cmake
# 设置可执行文件输出路径(必须在add_executable()之前指定)
set(EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/out/bin)

# 添加构建子目录
add_subdirectory(./module1)
add_subdirectory(./module2)

aux_source_directory(./ src_path)
# 使用指定源文件创建可执行文件
add_executable(CMakeDemo ${src_path})
# 设置可执行文件的前缀和后缀
set_target_properties(CMakeDemo PROPERTIES PREFIX "" SUFFIX ".run")

# 添加头文件搜索路径(main.c需要使用module1.h)
include_directories(${PROJECT_SOURCE_DIR}/src/module1)

# 链接可执行文件CMakeDemo与静态链接库module1_lib
target_link_libraries(CMakeDemo module1_lib)
```

&emsp;&emsp;`module1`下`CMakeLists.txt`内容：

```cmake
# 设置链接库文件输出路径(必须在add_library()之前指定)
set(LIBRARY_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/out/lib)

aux_source_directory(./ module1_src_path)
# 使用指定源文件创建静态(默认为静态)链接库
add_library(module1_lib ${module1_src_path})
# 设置链接库文件的前缀和后缀
set_target_properties(module1_lib PROPERTIES PREFIX "" SUFFIX ".lib")

# 添加头文件搜索路径(module1.c需要使用module2.h)
include_directories(${PROJECT_SOURCE_DIR}/src/module2)

# 链接静态链接库module1_lib和module2_lib
target_link_libraries(module1_lib module2_lib)
```

&emsp;&emsp;`module2`下`CMakeLists.txt`内容：

```cmake
# 设置链接库文件输出路径(必须在add_library()之前指定)
set(LIBRARY_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/out/lib)

aux_source_directory(./ module2_src_path)
# 使用指定源文件创建静态(默认为静态)链接库
add_library(module2_lib ${module2_src_path})
# 设置链接库文件的前缀和后缀
set_target_properties(module2_lib PROPERTIES PREFIX "" SUFFIX ".lib")
```

&emsp;&emsp;使用命令行构建工程：

```shell
# 提前在项目的根目录下创建build文件夹
cd /CMakeDemo/build
# 在build文件夹中运行cmake命令，构建生成的Makefile将存放在此文件夹中而不会污染工程
# cmake运行的路径中必须包含CMakeLists.txt文件
sudo cmake ../
# 使用make命令执行biuld中的Makefile完成编译
sudo make ./
# 运行bin下生成的可执行文件
sudo ../out/bin/CMakeDemo.run
# 清理Makefile
sudo rm -r ../build/*
```

&emsp;&emsp;以上工程示例中，使用`main.c`创建的可执行文件依赖于`module1`，`module1`依赖于`module2`，因此以上工程的编译链接过程为：

```xxx
1.编译module2.c生成静态链接库module2_lib
2.编译module1.c生成静态链接库module1_lib
3.链接静态链接库module1_lib与module2_lib
4.编译main.c生成可执行文件CMakeDemo
5.链接可执行文件CMakeDemo与静态链接库module1_lib
```

