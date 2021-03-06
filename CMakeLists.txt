cmake_minimum_required(VERSION 2.8)

set(PROJECT_NAME_STR libatlasclient)
set(CMAKE_MACOSX_RPATH 1)
project(${PROJECT_NAME_STR})


include_directories(3rd-party /usr/local/include)
link_directories(/usr/local/lib)

if(CMAKE_SYSTEM_NAME STREQUAL Linux)
    link_directories(/usr/lib/x86_64-linux-gnu)
endif()

# We need to initialize intern.cc first, since other files depend on it
# for interning strings directly or indirectly using static Tags
set(LIB_SOURCE_FILES
    util/intern.cc util/xxhash.c util/config.cc util/config_manager.cc util/environment.cc
    util/gzip.cc util/http.cc util/logger.cc util/strings.cc atlas_client.cc 
    meter/bucket_counter.cc meter/bucket_distribution_summary.cc meter/bucket_functions.cc
    meter/bucket_timer.cc meter/clock.cc meter/id.cc meter/interval_counter.cc meter/monotonic_counter.cc
    meter/percentile_buckets.cc meter/percentile_dist_summary.cc meter/percentile_timer.cc meter/statistic.cc
    meter/subscription_gauge.cc meter/subscription_long_task_timer.cc meter/subscription_manager.cc
    meter/subscription_registry.cc meter/subscription_timer.cc meter/validation.cc
    interpreter/aggregation.cc interpreter/all.cc interpreter/context.cc interpreter/expression.cc
    interpreter/group_by.cc interpreter/interpreter.cc interpreter/keep_drop_tags.cc
    interpreter/multiple_results.cc interpreter/query.cc interpreter/vocabulary.cc)

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -pedantic -Werror -Wall -std=c99 -fpic -D_GNU_SOURCE")
set(CMAKE_C_FLAGS_RELEASE "-O3 -march=native")

add_library(atlasclient SHARED ${LIB_SOURCE_FILES})
set_target_properties(atlasclient PROPERTIES COMPILE_FLAGS "${CMAKE_CXX_FLAGS} -Wextra -Wno-missing-braces")

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DCURL_STATICLIB -pedantic -Werror -fno-rtti -Wall -Wno-missing-braces -std=c++11")

if(CMAKE_SYSTEM_NAME STREQUAL Linux)
  configure_file(${CMAKE_SOURCE_DIR}/bundle/libcurl-linux.a libcurl.a COPYONLY)
  configure_file(${CMAKE_SOURCE_DIR}/bundle/libpcre-linux.a libpcre.a COPYONLY)
  add_library(atlasclientStatic STATIC ${LIB_SOURCE_FILES})
  set_target_properties(atlasclientStatic PROPERTIES OUTPUT_NAME atlasclient)
  target_link_libraries(atlasclientStatic ${CMAKE_BINARY_DIR}/libpcre.a ${CMAKE_BINARY_DIR}/libcurl.a z)
else()
  configure_file(${CMAKE_SOURCE_DIR}/bundle/libcurl-osx.a libcurl.a COPYONLY)
  configure_file(${CMAKE_SOURCE_DIR}/bundle/libpcre-osx.a libpcre.a COPYONLY)
endif()
target_link_libraries(atlasclient ${CMAKE_BINARY_DIR}/libpcre.a ${CMAKE_BINARY_DIR}/libcurl.a z )

set(APP_SOURCE_FILES main.cc)
add_executable(native_client ${APP_SOURCE_FILES})
set_target_properties(native_client PROPERTIES LINK_FLAGS "-Wl,-rpath,.")
target_link_libraries(native_client atlasclient pthread)

# test configuration
enable_testing()
file(GLOB TEST_SOURCE_FILES test/*.cc)

add_executable(runtests ${TEST_SOURCE_FILES})
set_target_properties(runtests PROPERTIES COMPILE_FLAGS "${CMAKE_CXX_FLAGS} -Wno-sign-compare")

add_library(gtest 3rd-party/src/gtest/gtest-all.cc)

configure_file(test/resources/metadata.json metadata.json COPYONLY)
configure_file(test/resources/subs1.json subs1.json COPYONLY)
configure_file(test/resources/many-subs.json many-subs.json COPYONLY)

target_link_libraries(runtests
        atlasclient
        pthread
        gtest)

add_test(runtests runtests)

install(TARGETS atlasclient DESTINATION lib)
install(FILES atlas_client.h types.h DESTINATION include/atlas)
install(DIRECTORY meter interpreter util DESTINATION include/atlas
        FILES_MATCHING PATTERN "*.h")
install(FILES resources/atlas-config-reference.json DESTINATION etc)

if(CMAKE_SYSTEM_NAME STREQUAL Linux)
  install(TARGETS atlasclientStatic DESTINATION lib)
  install(FILES ${CMAKE_SOURCE_DIR}/bundle/libcurl-linux.a DESTINATION lib RENAME libatlascurl.a)
  install(FILES ${CMAKE_SOURCE_DIR}/bundle/libpcre-linux.a DESTINATION lib RENAME libatlaspcre.a)
endif()
