

# CMake

假设目录结构如下

```c
project/
├── CMakeLists.txt
├── build/
├── include/
│   └── common_header.h
├── src/
│   ├── main1.c
│   ├── main2.c
│   └── shared.c
└── .vscode/
```

1.新建一个CMakeLists.txt文件

```cmake
# 设置版本
cmake_minimum_required(VERSION 3.10)

# 设置项目名称
project(MyProject)

# 添加头文件路径
include_directories(${PROJECT_SOURCE_DIR}/include)

# 添加共享的源文件
set(SHARED_SOURCES ${PROJECT_SOURCE_DIR}/src/shared.c)

# 为第一个程序创建可执行文件
add_executable(main1 ${PROJECT_SOURCE_DIR}/src/main1.c ${SHARED_SOURCES})

# 为第二个程序创建可执行文件
add_executable(main2 ${PROJECT_SOURCE_DIR}/src/main2.c ${SHARED_SOURCES})
```

```cmake
# 创建一个文件夹，用来存放cmake编译的一堆文件
mkdir build
cd build
# 因为CMakeLists.txt在上一层文件夹
cmake ..
make
```

## 变式

### 要是两个main程序的共享文件不一样

```cmake
cmake_minimum_required(VERSION 3.10)
project(MyProject)

# 添加头文件路径
include_directories(${PROJECT_SOURCE_DIR}/include)

# 为第一个程序指定源文件
set(SOURCES_MAIN1
    src/main1.c
    src/shared1.c
)

# 为第二个程序指定源文件
set(SOURCES_MAIN2
    src/main2.c
    src/shared2.c
)

# 为第一个程序创建可执行文件
add_executable(main1 ${SOURCES_MAIN1})

# 为第二个程序创建可执行文件
add_executable(main2 ${SOURCES_MAIN2})

# 如果需要链接库，可以使用 target_link_libraries
# target_link_libraries(main1 pthread)
# target_link_libraries(main2 pthread)
```

### 怎么把输出程序输出到特定的目录中

```cmake
cmake_minimum_required(VERSION 3.10)
project(MyProject)

# 添加头文件路径
include_directories(${PROJECT_SOURCE_DIR}/include)

# 设置输出目录
set(OUTPUT_DIR ${PROJECT_BINARY_DIR}/bin)

# 创建输出目录
file(MAKE_DIRECTORY ${OUTPUT_DIR})

# 为第一个程序指定源文件
set(SOURCES_MAIN1
    src/main1.c
    src/shared1.c
)

# 为第二个程序指定源文件
set(SOURCES_MAIN2
    src/main2.c
    src/shared2.c
)

# 为第一个程序创建可执行文件
add_executable(main1 ${SOURCES_MAIN1})

# 为第二个程序创建可执行文件
add_executable(main2 ${SOURCES_MAIN2})

# 设置可执行文件的输出目录
set_target_properties(main1 PROPERTIES RUNTIME_OUTPUT_DIRECTORY ${OUTPUT_DIR})
set_target_properties(main2 PROPERTIES RUNTIME_OUTPUT_DIRECTORY ${OUTPUT_DIR})

# 如果需要链接库，可以使用 target_link_libraries
# target_link_libraries(main1 pthread)
# target_link_libraries(main2 pthread)
```



## 样例

### 把输出程序输出到特定的目录中

![image-20250422155417513](D:/Code/TYPORA/Note/CMake.assets/image-20250422155417513.png)