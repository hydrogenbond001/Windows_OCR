
cmake_minimum_required(VERSION 3.15)
project(OCR LANGUAGES C CXX)

# ---------------------------------------------------------------------------
# 选项：是否启用 Unicode
option(USE_UNICODE "Build with UNICODE/_UNICODE defined" ON)
if(USE_UNICODE)
    add_compile_definitions(UNICODE _UNICODE)
endif()

if(MINGW)
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,--subsystem,console")
endif()

# ---------------------------------------------------------------------------
# 源文件：递归找所有 .cpp / .c
file(GLOB_RECURSE SRC_FILES
    # "${CMAKE_SOURCE_DIR}/src/write_rec/*.cpp"         //手写识别，效果不行，除非你写的跟印刷体一样。
    "${CMAKE_SOURCE_DIR}/src/screen_cut_rec/*.cpp"
    # opencv.cpp                                        //读取图片测试
    )

add_executable(OCR WIN32 ${SRC_FILES})

# -------- C++11 ----------
set_target_properties(OCR PROPERTIES
    CXX_STANDARD 11
    CXX_STANDARD_REQUIRED ON
)

add_definitions(-DNOMINMAX)

# ---------------------------------------------------------------------------
# 设置 OpenCV 路径（可被 -DOpenCV_DIR=xxx 覆盖）
if(NOT OpenCV_DIR)
    set(OpenCV_DIR "./opencv_mingw")  # 修改为你的路径
endif()

find_package(OpenCV REQUIRED)

if(NOT ncnn_DIR)
    set(ncnn_DIR "./ncnn_w64dev/lib/cmake/ncnn")  # 修改为你的路径
endif()
find_package(ncnn REQUIRED)
message(STATUS "ncnn_DIR = ${ncnn_DIR}")

target_include_directories(OCR
    PRIVATE                                   # 仅编译本目标时可见
        # ${OpenCV_INCLUDE_DIRS}
        "${CMAKE_SOURCE_DIR}/ncnn_w64dev/include/ncnn"
        # ${ncnn_INCLUDE_DIRS}
        "${CMAKE_SOURCE_DIR}/include"
)

target_link_libraries(OCR
    PRIVATE                                   # 只给 OCR 链接
    ncnn
    ${OpenCV_LIBS}
)

# ---------------------------------------------------------------------------
# 不同编译器 / 生成器的特定开关
if(MSVC)
    target_compile_options(OCR PRIVATE /utf-8)
else()  # GNU / Clang (MinGW)
    # -municode & -mwindows 只在 Windows GUI 程序有效
    target_link_options(OCR PRIVATE
        # $<$<BOOL:${USE_UNICODE}>:-municode>
        -mwindows
    )
endif()


# 构建完成后复制文件到 ../output/
add_custom_command(TARGET OCR POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E make_directory ../output

    # 复制所有 exe 文件
    COMMAND ${CMAKE_COMMAND} -E copy
        ${CMAKE_CURRENT_BINARY_DIR}/OCR.exe ../output

    # 复制 models 文件夹
    COMMAND ${CMAKE_COMMAND} -E copy_directory ${CMAKE_CURRENT_SOURCE_DIR}/models ../output/models

    # 复制 OpenCV 动态库
    COMMAND ${CMAKE_COMMAND} -E copy
        ${CMAKE_CURRENT_SOURCE_DIR}/opencv_mingw/x64/mingw/bin/libopencv_core4100.dll ../output/
    COMMAND ${CMAKE_COMMAND} -E copy
        ${CMAKE_CURRENT_SOURCE_DIR}/opencv_mingw/x64/mingw/bin/libopencv_highgui4100.dll ../output/
    COMMAND ${CMAKE_COMMAND} -E copy
        ${CMAKE_CURRENT_SOURCE_DIR}/opencv_mingw/x64/mingw/bin/libopencv_imgcodecs4100.dll ../output/
    COMMAND ${CMAKE_COMMAND} -E copy
        ${CMAKE_CURRENT_SOURCE_DIR}/opencv_mingw/x64/mingw/bin/libopencv_imgproc4100.dll ../output/
    COMMAND ${CMAKE_COMMAND} -E copy
        ${CMAKE_CURRENT_SOURCE_DIR}/opencv_mingw/x64/mingw/bin/libopencv_photo4100.dll ../output/
)