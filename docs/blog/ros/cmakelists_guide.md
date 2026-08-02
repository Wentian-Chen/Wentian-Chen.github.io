# ROS 包中的 CMakeLists.txt 详解

在 ROS1 的 catkin 体系中，每个功能包（package）的根目录下都有两个"身份证"文件：`package.xml` 负责声明这个包**是谁、依赖谁**（元信息），而 `CMakeLists.txt` 负责告诉编译系统**怎么把它编译出来**——要生成哪些可执行文件、链接哪些库、头文件去哪找。

简单说，`CMakeLists.txt` 是 CMake 的构建脚本。catkin 基于 CMake 构建，`catkin_make` / `catkin build` 本质上就是 CMake 的封装：它在工作空间根目录调用 CMake，而 CMake 会逐层读取每个包的 `CMakeLists.txt` 来生成最终的编译规则。所以理解 `CMakeLists.txt` 里几条核心命令（`find_package`、`catkin_package`、`add_executable`、`add_dependencies`、`target_link_libraries`），就基本掌握了 ROS 包构建的全部心智模型。

这篇文章是我在实践中踩过不少坑后的整理，重点讲清楚每个命令"是什么、为什么、什么时候用"，最后附一个完整的 Noetic 模板可以直接抄。

## 基础概念

### Package Configuration File

包配置文件通常命名为`YourPackageConfig.cmake` 或 `yourpackage-config.cmake`。它们由包的作者提供，包含了关于如何使用该包的所有必要信息，例如头文件路径、库文件路径以及它所依赖的其他包。或者名为 `FindYourPackage.cmake` 的文件。这些文件通常由用户或第三方编写，用于查找那些没有提供标准包配置文件的传统库。它们包含了一些启发式逻辑来定位库和头文件。

包配置文件的存储位置:如果使用安装类型的编译（即编译完一个包后，将相关文件安装到`your_workspace/install/your_package`），那么`*.cmake`通常会在`share/your_package/cmake` ，其中`your_workspace/install/your_package`称之为**包的路径前缀**(prefix)。

### setup.bash的作用

当终端启动后，执行工作空间的`setup.bash`后，会将工作空间内的**包的路径前缀**添加到变量`CMAKE_PREFIX_PATH`里。

确认包是否正确被添加的排查方法如下：

1. 确认包的路径前缀在变量`CMAKE_PREFIX_PATH`里，通常正常编译后都会存在。
2. 确认包存在于`your_workspace/install/your_package`,确定安装到`install`目录下。
3. 确认存在包配置文件，检查`share/your_package/cmake`里是否存在相关的`.cmake`文件。

!!! tip "技巧"
    很多"find_package 找不到包"的问题，本质都是没有 source 对应工作空间的 `setup.bash`。每次新开终端，先确认 `echo $CMAKE_PREFIX_PATH` 里能看到你的工作空间路径，再开始编译。

## find_package

### find_package的查找原理 

find_package 命令的查找过程相当复杂，但可以概括为两个主要模式：模块模式（Module Mode）和配置模式（Config Mode）。CMake 会按顺序尝试这两种模式来查找包。

1. 模块模式 (Module Mode)
      - 原理: CMake 自身或第三方开发者提供了`Find<PackageName>.cmake`脚本。这些脚本包含了查找特定库（如 OpenCV）所需头文件和库文件的逻辑。
      - 查找位置:
        - `CMAKE_MODULE_PATH`变量中指定的目录。
        - CMake 内置的模块目录，例如`/usr/share/cmake-<version>/Modules/`。
      - 如何确定: 如果`find_package(OpenCV)`成功，并且定义了 `OpenCV_INCLUDE_DIRS` 和 `OpenCV_LIBRARIES` 这样的变量，这说明使用了模块模式。

2. 配置模式 (Config Mode)

      - 原理: 库的开发者在安装时会生成 `<PackageName>Config.cmake` 或 `<PackageName>-config.cmake` 配置文件。这些文件包含了库的安装路径、头文件路径、库文件路径等信息。这种方式是现代 CMake 的推荐做法，因为它将配置信息与库本身一起安装，更加可靠。
      - 查找位置:
        - 在 `CMAKE_PREFIX_PATH` 变量中指定的目录。
        - 在环境变量中指定的目录，例如 `PATH`。
        - 在标准的系统路径下，如 `/usr/local/`、`/usr/` 等。
        - 在 ROS 中，catkin 会将工作区 (`devel` 或 `install` 目录) 自动添加到 `CMAKE_PREFIX_PATH`，这使得 `find_package` 能够找到 ROS 包。
      - 如何确定: 如果`find_package(Boost REQUIRED)` 成功，并且定义了 Boost::system 这样的导入目标，这意味使用了配置模式。导入目标是 CMake 推荐的现代链接方式，它将库的所有信息都封装在一个目标中。

### find_package的检索过程

当你调用 `find_package(<PackageName> REQUIRED)` 时：

- CMake 会遍历 `CMAKE_PREFIX_PATH` 中的每一个路径。
- 对于每个路径，它会尝试在以下结构中查找 `*Config.cmake` 文件： 
    - `<prefix>/share/<PackageName>/cmake/<PackageName>Config.cmake`
    - `<prefix>/lib/cmake/<PackageName>Config.cmake`
    - 等等。
- 一旦找到匹配的 `*Config.cmake` 文件，它就会加载该文件。这个文件会设置一系列变量（例如 `moveit_visual_tools_INCLUDE_DIRS`、`moveit_visual_tools_LIBRARIES`）并可能定义导入目标，从而使得你的项目能够链接到该包。

哪些包需要被添加到`find_package`: 如果你的程序使用了其他包的内容都需要写`find_package`

!!! warning "注意"
    `find_package` 只声明"我要用这个包"，它本身不负责链接。找到包后，还要在 `target_link_libraries` 里把对应的变量（如 `${catkin_LIBRARIES}`、`${OpenCV_LIBRARIES}`）真正链接到你的可执行文件上，不然编译能过、链接必挂。

### 手动添加查找目录

默认的查找路径有：

- `CMAKE_PREFIX_PATH`
- `/usr/local/lib`
- `/usr/lib`
- `/usr/local/include`
- `/usr/include`

手动在`CmakeLists.txt`中添加查找路径：
```cmake
# 设置 OpenCV 的安装路径# 将 /path/to/your/opencv/install 替换为您实际的安装路径
set(OpenCV_DIR "/path/to/your/opencv/install/lib/cmake/opencv4")  

# 或者使用 CMAKE_PREFIX_PATH，这会搜索指定目录下的 share/ 或 lib/cmake/ 子目录
set(CMAKE_PREFIX_PATH "/path/to/your/opencv/install")

# 亦或者
find_package(OpenCV REQUIRED
    HINTS "/path/to/opencv"
    "/another/path/to/search"
)
```

### 验证查找到的库
```cmake
# 打印出找到的变量，用于验证
message(STATUS "OpenCV include directories：${OpenCV_INCLUDE_DIRS}") 
message(STATUS "OpenCV libraries: ${OpenCV_LIBRARIES}")
```

!!! tip "技巧"
    编译前先用 `message(STATUS ...)` 把 `*_INCLUDE_DIRS` 和 `*_LIBRARIES` 打印出来，是排查"头文件找得到但链接报 undefined reference"这类问题最快的手段。

### 环境变量

`find_package`在寻找和导入包时会同时生成环境变量，比如`OpenCV_INCLUDE_DIRS`和`OpenCV_LIBRARIES`。
```cmake
# 这一步会生成catkin相关的变量
find_package(catkin REQUIRED COMPONENTS
    roscpp
    rospy
    # ...
)

# 此时会生成：
# - catkin_INCLUDE_DIRS
# - catkin_LIBRARIES
# - catkin_LIBRARY_DIRS
# - catkin_DEPENDS
```

同时，也支持在CMakeLists.txt中手动设置环境变量
```cmake
# catkin_INCLUDE_DIRS 包含了所有依赖包的头文件目录路径
# 例如可能包含：
# /opt/ros/noetic/include  # ROS核心头文件
# /usr/include/pcl-1.10    # PCL头文件
# /usr/include/opencv4     # OpenCV头文件

# 使用方式：
include_directories(
    ${catkin_INCLUDE_DIRS}  # 让编译器知道在哪里找头文件
)


# catkin_LIBRARIES 包含了所有依赖包的库文件列表
# 例如：
# libroscpp.so
# libroslib.so
# libopencv_core.so

# 使用方式：
target_link_libraries(你的程序名称
    ${catkin_LIBRARIES}  # 链接需要的库文件
)


# catkin_LIBRARY_DIRS 包含库文件所在的目录路径
# 例如：
# /opt/ros/noetic/lib
# /usr/local/lib

# 使用方式：
link_directories(
    ${catkin_LIBRARY_DIRS}  # 告诉链接器在哪里找库文件
)


# catkin_DEPENDS 记录了当前包依赖的其他catkin包
# 例如：roscpp rospy std_msgs

# 使用方式：
catkin_package(
    CATKIN_DEPENDS ${catkin_DEPENDS}  # 声明包的依赖关系
)
```

!!! note "说明"
    这四个变量是 `find_package(catkin REQUIRED COMPONENTS ...)` 的"副产品"。日常写包只需要记住两个：`include_directories(${catkin_INCLUDE_DIRS})` 管头文件，`target_link_libraries(... ${catkin_LIBRARIES})` 管库文件，剩下的交给 catkin 自动处理。

## add_dependencies

### 作用与使用场景

`add_dependencies(...)`的作用是在构建系统中建立目标之间的依赖关系。它主要用来确保一个目标（比如一个可执行文件或库）在另一个目标之前被构建。`add_dependencies()`关注的是编译时的依赖，即在编译一个目标之前，需要先完成哪些其他目标的编译。

`add_dependencies()`主要用于以下两种情况：

1. 代码生成和编译顺序: 当你的项目需要先生成某些文件（比如消息头文件、服务头文件）然后才能编译依赖这些文件的源代码时，你需要使`add_dependencies()`。
2. 强制编译顺序: 某些情况下，即使没有直接的代码依赖，你也可能需要强制 CMake 先编译某个目标，以满足特定的构建流程要求。

### 何时需要添加

你需要在以下情况中添加 `add_dependencies()`：

- ROS 消息和服务: 这是在 ROS 项目中最常见的使用场景。当你的节点使用了自定义的消息或服务文件时，catkin 需要先生成这些文件的 C++ 或 Python 头文件。你需要确保在编译你的节点之前，这些头文件已经被生成。
- 与其他目标存在非代码依赖: 如果一个目标的生成需要另一个目标的输出文件。

```cmake
# 一个典型场景：节点依赖自定义消息的生成
add_executable(my_node src/my_node.cpp)
add_dependencies(my_node ${${PROJECT_NAME}_EXPORTED_TARGETS} ${catkin_EXPORTED_TARGETS})
```

!!! note "说明"
    模板里常见的 `add_dependencies(${PROJECT_NAME}_node ${${PROJECT_NAME}_EXPORTED_TARGETS} ${catkin_EXPORTED_TARGETS})` 就是在做这件事：把"消息生成目标"（EXPORTED_TARGETS）挂在你的节点目标前面，保证自定义消息的头文件先被生成。

## target_link_libraries

`target_link_libraries`的作用是链接库文件到可执行文件或库文件。它的主要目的是告诉编译器在编译你的程序时需要使用哪些已经编译好的库，以便程序可以调用这些库中定义的函数和类。`target_link_libraries()` 关注的是链接时的依赖，即程序运行时需要哪些库文件。

`target_link_libraries` 主要用于以下几个方面：

1. 链接 ROS 库: 当你的节点需要使用 ROS 提供的功能（例如发布/订阅话题、调用服务等）时，你需要链接 ROS 相关的库。例如，rospy、roscpp、message_generation 等。
2. 链接外部库: 如果你的程序使用了非 ROS 的第三方库，比如 OpenCV、PCL 等，你也需要使用 `target_link_libraries` 来链接这些库，这样你的程序才能正常调用它们提供的功能。
3. 链接自定义库: 当你的项目中包含自己编写的库（add_library 创建的），并且你想在另一个可执行文件或库中使用它时，同样需要使用 `target_link_libraries` 来进行链接。

需要在以下情况中添加 target_link_libraries：

- 创建可执行文件: 当你使用`add_executable`创建一个可执行文件时，如果这个节点需要依赖任何库，你都必须在 `target_link_libraries` 中指定这些库。
- 创建库: 当你使用 `add_library` 创建一个库时，如果这个库需要依赖其他库，你也需要使用 `target_link_libraries` 来链接它们。

### 链接ROS库
```cmake
# 创建一个名为 "my_ros_node" 的可执行文件
add_executable(my_ros_node src/my_ros_node.cpp)

# 链接 ROS 和 catkin 相关的库
target_link_libraries(my_ros_node
  ${catkin_LIBRARIES}
)
```

`catkin_LIBRARIES` 是一个由 `catkin_make` 自动生成的变量，它包含了所有在 `package.xml` 文件中通过 `<depend>`标签声明的 ROS 依赖项。你无需手动列出 `roscpp`、`message_generation` 等库，catkin 会为你处理。你只需确保你的 `package.xml` 文件中的依赖项是正确的。

### 链接外部库
```cmake
# 找到 OpenCV 包
find_package(OpenCV REQUIRED)

# 创建一个名为 "image_processor" 的可执行文件
add_executable(image_processor src/image_processor.cpp)

# 链接 ROS 库和 OpenCV 库
target_link_libraries(image_processor
  ${catkin_LIBRARIES}
  ${OpenCV_LIBRARIES}
)
```

在`target_link_libraries`填写库的路径。确定库的变量或者路径的方法如下：

1. 使用 `find_package()`： 首先，在 `CMakeLists.txt` 中使用 `find_package(OpenCV REQUIRED)` 来找到并加载 OpenCV 库。
2. 查看 `find_package` 文档： 几乎所有主流库的 `find_package` 模块都会定义一个或多个变量，其中通常会有一个 `_LIBRARIES` 变量，包含了链接所需的库文件路径。对于 OpenCV，这个变量就是 `OpenCV_LIBRARIES`。
3. 检查库的安装路径： 如果 `find_package` 失败，你需要确保该库已经正确安装在你的系统上。你可以在 `/usr/lib/` 或 `/usr/local/lib/` 目录下查找库文件的名称（例如 `libopencv_core.so`）。

### 链接自定义库
```cmake
# 1. 创建自定义库
add_library(common_utils src/common_utils.cpp)

# 2. 创建一个名为 "main_app" 的可执行文件
add_executable(main_app src/main_app.cpp)

# 3. 链接 ROS 库和自定义库
target_link_libraries(main_app
  ${catkin_LIBRARIES}
  common_utils
)
```

配置方法：

1. 使用 `add_library()` 定义： 当你使用 `add_library(common_utils ...)` 创建库时，`common_utils` 就是这个库在 `CMakeLists.txt` 中的逻辑名称。
2. 直接引用： 在 `target_link_libraries` 中，你可以直接使用这个逻辑名称来链接你的自定义库。CMake 会自动处理库文件的路径和依赖关系。

## 一个完整的 catkin CMakeLists.txt 模板（ROS1 Noetic）

最后给一个可以直接抄的模板：一个最简单的 roscpp 节点包（含 talker/listener 两个可执行文件），对应 ROS wiki 上 beginner_tutorials 的标准写法。新建包时用 `catkin_create_pkg` 生成的文件骨架和这个几乎一模一样。

```cmake
cmake_minimum_required(VERSION 3.0.2)
project(ros_tutorials_beginner)

## 指定 C++ 标准，避免编译器默认标准过低
set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

## 声明依赖的 catkin 组件：rospy/roscpp 是 C++ 节点必需，std_msgs 是基础消息包
find_package(catkin REQUIRED COMPONENTS
  roscpp
  std_msgs
)

## 声明本包对外导出的信息（本包如果有头文件/库给其他包用才需要写完整）
catkin_package(
  INCLUDE_DIRS include
  LIBRARIES ros_tutorials_beginner
  CATKIN_DEPENDS roscpp std_msgs
)

## 头文件搜索路径：先找本包的 include，再找所有依赖包的头文件
include_directories(
  include
  ${catkin_INCLUDE_DIRS}
)

## 添加可执行文件（每个节点一个 add_executable + 一对 add_dependencies / target_link_libraries）
add_executable(talker src/talker.cpp)
add_dependencies(talker ${${PROJECT_NAME}_EXPORTED_TARGETS} ${catkin_EXPORTED_TARGETS})
target_link_libraries(talker ${catkin_LIBRARIES})

add_executable(listener src/listener.cpp)
add_dependencies(listener ${${PROJECT_NAME}_EXPORTED_TARGETS} ${catkin_EXPORTED_TARGETS})
target_link_libraries(listener ${catkin_LIBRARIES})
```

!!! warning "注意"
    如果包的 `package.xml` 里没有声明 `<depend>roscpp</depend>` 等依赖，而 `CMakeLists.txt` 里却写了，编译时 catkin 会直接报错说依赖不一致。**`package.xml` 和 `CMakeLists.txt` 里的依赖列表必须对齐**，这是新手最容易忽略的一点。

对应的 `package.xml` 大致长这样（仅列出关键标签）：

```xml
<package format="2">
  <name>ros_tutorials_beginner</name>
  <version>0.0.0</version>
  <description>a simple roscpp package</description>
  <maintainer email="you@example.com">you</maintainer>
  <license>Apache-2.0</license>

  <buildtool_depend>catkin</buildtool_depend>
  <depend>roscpp</depend>
  <depend>std_msgs</depend>
</package>
```

把上面的文件放进 `src/` 下的包目录，在 `src/talker.cpp`、`src/listener.cpp` 里写好自己的节点代码，回到工作空间根目录 `catkin_make`，一个最小的 ROS1 节点包就算跑通了。
