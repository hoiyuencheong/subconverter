cmake_minimum_required(VERSION 3.4)
project(subconverter LANGUAGES CXX)

set(BUILD_TARGET_NAME ${PROJECT_NAME})
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake/")
include_directories("${CMAKE_SOURCE_DIR}/include/")

if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release)
endif()
set(CMAKE_CXX_STANDARD 20)

if(NOT MSVC)
    add_compile_options(-Wall -Wextra -Wno-unused-parameter -Wno-unused-result)
else()
    add_compile_options(/W4)
endif()

# 移除 std::regex 支持
option(USING_MALLOC_TRIM "Call malloc_trim after processing request to lower memory usage (Your system must support malloc_trim)." OFF)
option(BUILD_STATIC_LIBRARY "Build a static library containing only the essential part." OFF)

include(CheckCXXSourceCompiles)
check_cxx_source_compiles("
#include<string>
int main(){std::to_string(0);}
" HAVE_TO_STRING)

if(NOT BUILD_STATIC_LIBRARY)
    add_executable(${BUILD_TARGET_NAME} 
        src/generator/config/nodemanip.cpp
        src/generator/config/ruleconvert.cpp
        src/generator/config/subexport.cpp
        src/generator/template/templates.cpp
        src/handler/interfaces.cpp
        src/handler/multithread.cpp
        src/handler/upload.cpp
        src/handler/webget.cpp
        src/handler/settings.cpp
        src/main.cpp
        src/parser/infoparser.cpp
        src/parser/subparser.cpp
        src/script/cron.cpp
        src/script/script_quickjs.cpp
        src/server/webserver_httplib.cpp
        src/utils/base64/base64.cpp
        src/utils/codepage.cpp
        src/utils/file.cpp
        src/utils/logger.cpp
        src/utils/md5/md5.cpp
        src/utils/network.cpp
        src/utils/regexp.cpp
        src/utils/string.cpp
        src/utils/system.cpp
        src/utils/urlencode.cpp)

    target_include_directories(${BUILD_TARGET_NAME} PRIVATE src)
    
    # 检测操作系统和架构
    if(APPLE)
        if(CMAKE_SYSTEM_PROCESSOR MATCHES "arm64")
            # MacOS (M系列芯片)
            target_include_directories(${BUILD_TARGET_NAME} PRIVATE /opt/homebrew/include)
            message(STATUS "Detected MacOS with Apple Silicon (arm64)")
        else()
            # MacOS (Intel)
            target_include_directories(${BUILD_TARGET_NAME} PRIVATE /usr/local/include)
            message(STATUS "Detected MacOS with Intel (x86_64)")
        endif()
    elseif(UNIX)
        # Linux
        target_include_directories(${BUILD_TARGET_NAME} PRIVATE /usr/local/include)
        message(STATUS "Detected Linux")
    endif()
    

    target_link_directories(${BUILD_TARGET_NAME} PRIVATE ${CMAKE_SOURCE_DIR})

    find_package(PkgConfig REQUIRED)

    set(THREADS_PREFER_PTHREAD_FLAG ON)
    find_package(Threads REQUIRED)
    target_link_libraries(${BUILD_TARGET_NAME} ${CMAKE_THREAD_LIBS_INIT})

    find_package(CURL 7.54.0 REQUIRED)
    target_include_directories(${BUILD_TARGET_NAME} PRIVATE ${CURL_INCLUDE_DIRS})
    target_link_directories(${BUILD_TARGET_NAME} PRIVATE ${CURL_LIBRARY_DIRS}) 
    target_link_libraries(${BUILD_TARGET_NAME} CURL::libcurl)
    target_compile_definitions(${BUILD_TARGET_NAME} PRIVATE -DCURL_STATICLIB)

    if(APPLE)
    elseif(UNIX)
        pkg_check_modules(NGHTTP2 REQUIRED libnghttp2)
        target_link_directories(${BUILD_TARGET_NAME} PRIVATE /usr/local/nghttp2/lib)
        target_link_libraries(${BUILD_TARGET_NAME} ${NGHTTP2_LIBRARIES})
        pkg_check_modules(BROTLI REQUIRED libbrotlidec libbrotlienc libbrotlicommon)
        target_link_libraries(${BUILD_TARGET_NAME} ${BROTLI_LIBRARIES})
    endif()

    # 查找并链接 ZLIB
    find_package(ZLIB REQUIRED)
    target_include_directories(${BUILD_TARGET_NAME} PRIVATE ${ZLIB_INCLUDE_DIRS})
    target_link_libraries(${BUILD_TARGET_NAME} ZLIB::ZLIB)

    # 查找并链接 OpenSSL
    find_package(OpenSSL REQUIRED)
    target_include_directories(${BUILD_TARGET_NAME} PRIVATE ${OPENSSL_INCLUDE_DIR})
    target_link_libraries(${BUILD_TARGET_NAME} OpenSSL::SSL OpenSSL::Crypto)

    # 查找并链接 RapidJSON
    find_package(Rapidjson REQUIRED)
    target_include_directories(${BUILD_TARGET_NAME} PRIVATE ${RAPIDJSON_INCLUDE_DIRS})

    # 查找并链接 toml11
    find_package(toml11 REQUIRED)
    target_include_directories(${BUILD_TARGET_NAME} PRIVATE ${TOML11_INCLUDE_DIRS})

    # 查找并链接 yaml-cpp
    pkg_check_modules(YAML_CPP yaml-cpp>=0.6.3 REQUIRED)
    find_library(YAML_CPP_LIBRARY NAMES yaml-cpp yaml-cppd PATHS ${YAML_CPP_LIBRARY_DIRS})
    target_include_directories(${BUILD_TARGET_NAME} PRIVATE ${YAML_CPP_INCLUDE_DIRS})
    target_link_libraries(${BUILD_TARGET_NAME} ${YAML_CPP_LIBRARY})
    target_compile_definitions(${BUILD_TARGET_NAME} PRIVATE -DYAML_CPP_STATIC_DEFINE)

    # 查找并链接 PCRE2
    find_package(PCRE2 REQUIRED)
    target_include_directories(${BUILD_TARGET_NAME} PRIVATE ${PCRE2_INCLUDE_DIRS})
    target_link_libraries(${BUILD_TARGET_NAME} ${PCRE2_LIBRARY})
    target_compile_definitions(${BUILD_TARGET_NAME} PRIVATE -DPCRE2_STATIC)

     # 查找并链接 QuickJS
    find_package(QuickJS REQUIRED)
    target_include_directories(${BUILD_TARGET_NAME} PRIVATE ${QUICKJS_INCLUDE_DIRS})
    # 链接 QuickJS 和 dl 
    target_link_libraries(${BUILD_TARGET_NAME} ${QUICKJS_LIBRARIES} dl)

    # 查找并链接 LibCron
    find_package(LibCron REQUIRED)
    target_include_directories(${BUILD_TARGET_NAME} PRIVATE ${LIBCRON_INCLUDE_DIRS})
    target_link_libraries(${BUILD_TARGET_NAME} ${LIBCRON_LIBRARIES})

    if(WIN32)
        target_link_libraries(${BUILD_TARGET_NAME} wsock32 ws2_32)
    else()
        include(GNUInstallDirs)
        install(TARGETS ${BUILD_TARGET_NAME} DESTINATION ${CMAKE_INSTALL_BINDIR}/${BUILD_TARGET_NAME})
        install(DIRECTORY base/ DESTINATION ${CMAKE_INSTALL_BINDIR}/${BUILD_TARGET_NAME} FILES_MATCHING PATTERN "*")
    endif()

else() # BUILD_STATIC_LIBRARY
    add_library(${BUILD_TARGET_NAME} STATIC
        src/generator/config/ruleconvert.cpp
        src/generator/config/subexport.cpp
        src/generator/template/templates.cpp
        src/lib/wrapper.cpp
        src/parser/subparser.cpp
        src/utils/base64/base64.cpp
        src/utils/codepage.cpp
        src/utils/logger.cpp
        src/utils/md5/md5.cpp
        src/utils/network.cpp
        src/utils/regexp.cpp
        src/utils/string.cpp
        src/utils/urlencode.cpp)

    target_compile_definitions(${BUILD_TARGET_NAME} PRIVATE -DNO_JS_RUNTIME -DNO_WEBGET)
    target_include_directories(${BUILD_TARGET_NAME} PUBLIC src)

    find_package(Rapidjson REQUIRED)
    target_include_directories(${BUILD_TARGET_NAME} PRIVATE ${RAPIDJSON_INCLUDE_DIRS})

    find_package(PkgConfig REQUIRED)

    pkg_check_modules(YAML_CPP yaml-cpp>=0.6.3 REQUIRED)
    find_library(YAML_CPP_LIBRARY NAMES yaml-cpp yaml-cppd PATHS ${YAML_CPP_LIBRARY_DIRS})
    target_include_directories(${BUILD_TARGET_NAME} PRIVATE ${YAML_CPP_INCLUDE_DIRS})
    target_link_libraries(${BUILD_TARGET_NAME} PRIVATE ${YAML_CPP_LIBRARY})
    target_compile_definitions(${BUILD_TARGET_NAME} PRIVATE -DYAML_CPP_STATIC_DEFINE)

    find_package(PCRE2 REQUIRED)
    target_include_directories(${BUILD_TARGET_NAME} PRIVATE ${PCRE2_INCLUDE_DIRS})
    target_link_libraries(${BUILD_TARGET_NAME} PRIVATE ${PCRE2_LIBRARY})
    target_compile_definitions(${BUILD_TARGET_NAME} PRIVATE -DPCRE2_STATIC)

    if(WIN32)
        target_link_libraries(${BUILD_TARGET_NAME} PRIVATE ws2_32)
    endif()

endif() # BUILD_STATIC_LIBRARY

if(HAVE_TO_STRING)
    target_compile_definitions(${BUILD_TARGET_NAME} PRIVATE -DHAVE_TO_STRING)
endif()

if(USING_MALLOC_TRIM)
    target_compile_definitions(${BUILD_TARGET_NAME} PRIVATE -DMALLOC_TRIM)
endif()