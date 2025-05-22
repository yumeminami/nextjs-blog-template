---
title: CMake 编译安装导出 Package
date: 2025-05-22T21:10:00+08:00
updated: 2025-05-22T21:10:00+08:00
keywords: ["cmake", "install", "export"]
featured: true
summary: "如何使用 CMake install export 让我们的目标可以被 find_pacakge 使用"
---

## 背景

在我司的机器人项目中，所有实现都是基于 `ros` 的，但是有客户需求是，只需要机器人的 `harware` 部分，不需要 `ros` 的部分，因此需要将 `ros` 和 `hardware` 进行解耦，并且将 `hareware` 打包分发。

## 解决方案

最终的目标是只要在 `CMakeLists.txt` 中添加 `find_package` 就可以找到我们编译的库 以及 `target_link_libraries` 就可以链接我们编译的库。

1. 修改 `hardware` 以及所有子库的 `CMakeLists.txt` 文件中的 `target_include` 和 `target_link`
    1. **BUILD_INTERFACE 与 INSTALL_INTERFACE**

    ```C++
    project(hardware)
    target_include_directories(hardware
          PUBLIC
              $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
              $<INSTALL_INTERFACE:include/${PROJECT_NAME}/include>
    )
    target_link_libraries(hardware
        PUBLIC
            $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/lib/lib1.so>
            $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/lib/lib2.so>
            $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/lib/lib3.so>
            $<INSTALL_INTERFACE:lib1.so>
            $<INSTALL_INTERFACE:lib2.so>
            $<INSTALL_INTERFACE:lib3.so>
    )
    ```

    - **BUILD_INTERFACE**：表示编译时需要的路径
    - **INSTALL_INTERFACE**：表示被安装后需要的路径

2. 修改 `hareware` 以及所有子库的 `CMakeLists.txt` 文件中的 `install` 命令, 对齐所有 `$<INSTALL_INTERFACE>` 的路径, 例如

    ```C++
      install(DIRECTORY 
      ${CMAKE_CURRENT_SOURCE_DIR}/include
      DESTINATION include/${PROJECT_NAME})

      install(FILES
        ${CMAKE_CURRENT_SOURCE_DIR}/lib/lib1.so
        ${CMAKE_CURRENT_SOURCE_DIR}/lib/lib2.so
        ${CMAKE_CURRENT_SOURCE_DIR}/lib/lib3.so
        DESTINATION lib
      )
    ```

3. 安装导出，由于子库并不需要独立分发，因此只需要在顶层 `hardware` 的 `CMakeLists.txt` 文件中添加 `install(TARGETS ... EXPORT <export_name>)` 命令

    ```bash
    project(hardware)

    install(TARGETS ${PROJECT_NAME}
            lib1
            lib2
            lib3
            EXPORT ${PROJECT_NAME}Targets
            RUNTIME DESTINATION bin
            LIBRARY DESTINATION lib
            ARCHIVE DESTINATION lib
            INCLUDES DESTINATION include
    )

    include(CMakePackageConfigHelpers)
    write_basic_package_version_file(
        "${PROJECT_NAME}ConfigVersion.cmake"
        VERSION ${PROJECT_VERSION}
        COMPATIBILITY SameMajorVersion
    )

    install(FILES
        "${PROJECT_BINARY_DIR}/${PROJECT_NAME}Config.cmake"
        "${PROJECT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake"
        DESTINATION lib/cmake/${PROJECT_NAME}
    )

    # Install the export set
    install(EXPORT ${PROJECT_NAME}Targets
        FILE ${PROJECT_NAME}Targets.cmake
        NAMESPACE ${PROJECT_NAME}::
        DESTINATION lib/cmake/${PROJECT_NAME}
    )
    ```

4. 添加 `cmake.config` 文件到项目 `cmake` 目录下

    file name: `hardwareConfig.cmake`

    ```bash
    @PACKAGE_INIT@

    include("${CMAKE_CURRENT_LIST_DIR}/${PROJECT_NAME}Targets.cmake")
    ```

5. 编译安装

```bash
cd <project>
mkdir build && cd build
cmake .. -DCMAKE_INSTALL_PREFIX=<install_prefix>
sudo make -j$(nproc) install
```

`<install_prefix>` 目录就是我们可以分发目标了, 使用方通过 `find_package(hardware REQUIRED)` 时，可以得到`hareware::hareware` 以及子库`hardware::lib1` 和 `hardware::lib2` 和 `hardware::lib3` 三个目标的所有信息

## 如何使用

1. 小工具

由于打包好的库，我们可能从仓库克隆下来使用，而不是安装到我们的用户目录或者系统目录当中(`/usr/include/`，`/usr/local/include`) 中, 当我们 `find_package` 时会找不到，因此我借鉴了 `ros` 的机制。

在 `<install_prefix>` 目录下提供一个脚本:

```bash
#!/usr/bin/env sh

script_dir=$(cd "$(dirname "$0")" && pwd)

export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$script_dir/lib
export PATH=$PATH:$script_dir/bin
export CMAKE_PREFIX_PATH=$script_dir:$CMAKE_PREFIX_PATH
```

用户 `source` 这个脚本后就可以使用 `find_package` 找到我们分发的库了。

## Reference Links

- https://cmake.org/cmake/help/latest/guide/importing-exporting/index.html
- https://stackoverflow.com/questions/55635294/how-to-create-packages-with-cmake
- https://blog.csdn.net/qq_21438461/article/details/145266766
- https://blog.csdn.net/csy1021/article/details/139182662
