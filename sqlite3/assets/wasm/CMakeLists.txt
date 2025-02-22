cmake_minimum_required(VERSION 3.14)
set(PROJECT_NAME "sqlite3_web")
project(${PROJECT_NAME} LANGUAGES C)

include(FetchContent)
FetchContent_Declare(
  sqlite3
  URL https://sqlite.org/2022/sqlite-autoconf-3380200.tar.gz
)
FetchContent_MakeAvailable(sqlite3)

set(wasm_visibility "__attribute__((visibility(\"default\")))")

macro(base_sqlite3_target name)
    add_executable(${name}
        "${sqlite3_SOURCE_DIR}/sqlite3.c"
        os_web.c
        helpers.c
    )

    target_link_options(${name} PRIVATE -nostartfiles -Wl,--import-memory -Wl,--no-entry -Wl,--export-dynamic)
    target_include_directories(${name} PRIVATE "${PROJECT_SOURCE_DIR}/")
    target_include_directories(${name} PRIVATE ${sqlite3_SOURCE_DIR})
    target_compile_definitions(${name} PRIVATE
        _HAVE_SQLITE_CONFIG_H
        SQLITE_API=${wasm_visibility}
    )
endmacro()

base_sqlite3_target(sqlite3_debug)
file(DOWNLOAD https://raw.githubusercontent.com/sqlite/sqlite/master/src/test_vfstrace.c vfstrace.c)
target_sources(sqlite3_debug PRIVATE vfstrace.c)
target_compile_options(sqlite3_debug PRIVATE -g)
target_compile_definitions(sqlite3_debug PRIVATE SQLITE_ENABLE_VFSTRACE)
set_target_properties(sqlite3_debug PROPERTIES OUTPUT_NAME "sqlite3" SUFFIX ".debug.wasm")

base_sqlite3_target(sqlite3_opt)
target_compile_options(sqlite3_opt PRIVATE -Ofast)
set_target_properties(sqlite3_opt PROPERTIES OUTPUT_NAME "sqlite3" SUFFIX ".tmp.wasm")
add_custom_command(TARGET sqlite3_opt POST_BUILD
    COMMAND wasm-opt sqlite3.tmp.wasm -O4 -o sqlite3.wasm
)

add_custom_target(output)
add_custom_command(TARGET output COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_BINARY_DIR}/sqlite3.wasm ${PROJECT_SOURCE_DIR}/../../example/web/sqlite3.wasm)
add_custom_command(TARGET output COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_BINARY_DIR}/sqlite3.debug.wasm ${PROJECT_SOURCE_DIR}/../../example/web/sqlite3.debug.wasm)
add_dependencies(output sqlite3_debug sqlite3_opt)
