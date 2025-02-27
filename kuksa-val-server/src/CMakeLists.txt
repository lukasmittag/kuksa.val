#
# ******************************************************************************
# Copyright (c) 2018-2022 Robert Bosch GmbH and others.
#
# All rights reserved. This configuration file is provided to you under the
# terms and conditions of the Eclipse Distribution License v1.0 which
# accompanies this distribution, and is available at
# http://www.eclipse.org/org/documents/edl-v10.php
#
# *****************************************************************************

project(kuksa-val-server)
enable_testing()

######
# CMake configuration responsible for building core library and executable

###
# Setup target names
set(SERVER_EXE_NAME "${PROJECT_NAME}" )
set(SERVER_OBJ_LIB_NAME ${SERVER_EXE_NAME}-object )
set(SERVER_LIB_NAME ${SERVER_EXE_NAME}-core )
set(SERVER_LIB_STATIC_NAME ${SERVER_LIB_NAME}-static )
set(GRPC_GENERATED_LIB_NAME kuksa_viss_grpc_generated-object) 
set(GRPC_CLIENT_EXE_NAME kuksa_viss_grpc_client) 


### 
# Setup grpc
set(proto_gen_dir "${CMAKE_BINARY_DIR}/proto")
set(kuksa_proto "${CMAKE_CURRENT_SOURCE_DIR}/../protos/kuksa.proto")
get_filename_component(kuksa_proto_path "${kuksa_proto}" PATH)
file(MAKE_DIRECTORY ${proto_gen_dir})
protobuf_gen(PROTO ${kuksa_proto}
    PROTO_PATH ${kuksa_proto_path}
    PROTO_INC ${_PROTO_INC}
    OUTPUT ${proto_gen_dir}
)
include_directories("${proto_gen_dir}")
add_library(${GRPC_GENERATED_LIB_NAME} OBJECT ${PROTO_SRCS}
  ${GRPC_SRCS}
)
target_link_libraries(${GRPC_GENERATED_LIB_NAME}
  PUBLIC
    ${_GRPC_GRPCPP}
    ${_PROTOBUF_LIBPROTOBUF}
    ${_REFLECTION}
)


###
# Setup command-line options

# Set if executable should be built
set(BUILD_EXE ON CACHE BOOL "Build '${SERVER_EXE_NAME}' executable")
# Use address sanitizer when building with Clang
set(ADDRESS_SAN OFF CACHE BOOL "Build with AddressSanitizer")

###
# Define source list
file(GLOB LIB_SOURCES *.cpp)
list(REMOVE_ITEM LIB_SOURCES "${PROJECT_SOURCE_DIR}/main.cpp")

###
# set main 'object library' as target to prevent multiple compilations of same sources
add_library(${SERVER_OBJ_LIB_NAME} OBJECT ${LIB_SOURCES})

target_compile_features(${SERVER_OBJ_LIB_NAME} PUBLIC cxx_std_14)

###
# Setup object library
target_include_directories(${SERVER_OBJ_LIB_NAME}
  PUBLIC
    $<INSTALL_INTERFACE:include>
    $<INSTALL_INTERFACE:include/interface>
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/../include>
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/../include/interface>
)

target_link_libraries(${SERVER_OBJ_LIB_NAME}
  PUBLIC
    ${GRPC_GENERATED_LIB_NAME}
)

if(CMAKE_COMPILER_IS_GNUCXX)
  target_compile_options(${SERVER_OBJ_LIB_NAME} PUBLIC -Wall -Wextra -Werror -Wno-error=unused-result -Wno-error=unused-parameter -Wno-error=deprecated-declarations -Wno-error=deprecated-copy -Wno-error=nonnull)
else(CMAKE_COMPILER_IS_GNUCXX)
  target_compile_options(${SERVER_OBJ_LIB_NAME} PUBLIC -Wall -Wextra -Werror -Wno-error=parentheses)
endif(CMAKE_COMPILER_IS_GNUCXX)

IF(CMAKE_BUILD_TYPE MATCHES Debug)
  message(STATUS "This is a DEBUG build")
  target_compile_options(${SERVER_OBJ_LIB_NAME} PUBLIC -g -O0)
ELSEIF(CMAKE_BUILD_TYPE MATCHES Coverage)
  if("${CMAKE_C_COMPILER_ID}" MATCHES "(Apple)?[Cc]lang" OR "${CMAKE_CXX_COMPILER_ID}" MATCHES "(Apple)?[Cc]lang")
    target_compile_options(${SERVER_OBJ_LIB_NAME} PUBLIC -g -O0 -fprofile-instr-generate -fcoverage-mapping)
    target_link_libraries(${SERVER_OBJ_LIB_NAME} PUBLIC -fprofile-instr-generate -fcoverage-mapping)
  elseif(CMAKE_COMPILER_IS_GNUCXX)
    target_compile_options(${SERVER_OBJ_LIB_NAME} PUBLIC -g -O0 --coverage -fprofile-arcs -ftest-coverage)
    target_link_libraries(${SERVER_OBJ_LIB_NAME} PUBLIC gcov)
  else()
    message(FATAL_ERROR "Code coverage requires Clang or GCC. Aborting.")
  endif()
ENDIF(CMAKE_BUILD_TYPE MATCHES Debug)

if(BUILD_UNIT_TEST)
  target_compile_definitions(${SERVER_OBJ_LIB_NAME} PUBLIC UNIT_TEST)
endif(BUILD_UNIT_TEST)

# link library dependencies
# Setting max stack size explicetely to current default value of glibc on Ubuntu, to force MUSL
# builds using the same max. Otherwise you might have hard to debug differences between MUSL and
# builds.
# See also https://wiki.musl-libc.org/functional-differences-from-glibc.html#Thread-stack-size
target_link_libraries(${SERVER_OBJ_LIB_NAME}  PUBLIC jwt-cpp jsonpath jsoncons ${CMAKE_THREAD_LIBS_INIT} -Wl,-z,stack-size=8388608)

if ("${ADDRESS_SAN}" STREQUAL "ON" AND "${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
  target_compile_options(${SERVER_OBJ_LIB_NAME} PUBLIC -g -fsanitize=address -fno-omit-frame-pointer -DGRPC_BUILD_WITH_BORING_SSL_ASM=0)
  target_link_libraries(${SERVER_OBJ_LIB_NAME} PUBLIC "-g -fsanitize=address ")
endif()

###
# Define common libraries

# shared and static libraries built from the same object files
add_library(${SERVER_LIB_NAME}        SHARED $<TARGET_OBJECTS:${SERVER_OBJ_LIB_NAME}>)
add_library(${SERVER_LIB_STATIC_NAME} STATIC $<TARGET_OBJECTS:${SERVER_OBJ_LIB_NAME}>)
# re-use include directories from object library
target_include_directories(${SERVER_LIB_NAME}        PUBLIC $<TARGET_PROPERTY:${SERVER_OBJ_LIB_NAME},INTERFACE_INCLUDE_DIRECTORIES>)
target_include_directories(${SERVER_LIB_STATIC_NAME} PUBLIC $<TARGET_PROPERTY:${SERVER_OBJ_LIB_NAME},INTERFACE_INCLUDE_DIRECTORIES>)
# re-use link libraries from object library
target_link_libraries(${SERVER_LIB_NAME}        PUBLIC $<TARGET_PROPERTY:${SERVER_OBJ_LIB_NAME},INTERFACE_LINK_LIBRARIES>)
target_link_libraries(${SERVER_LIB_STATIC_NAME} PUBLIC $<TARGET_PROPERTY:${SERVER_OBJ_LIB_NAME},INTERFACE_LINK_LIBRARIES>)


if (ENABLE_COVERAGE)
  add_coverage(${SERVER_OBJ_LIB_NAME})
endif()



if(BUILD_EXE)
  add_executable(${SERVER_EXE_NAME} main.cpp
  )

  if (ENABLE_COVERAGE)
    add_coverage(${SERVER_EXE_NAME})
  endif()

  # using static library to simplify existing packaging and maintainance
  target_link_libraries(${SERVER_EXE_NAME} PRIVATE ${SERVER_LIB_STATIC_NAME})

  # link executable dependencies
  target_link_libraries(${SERVER_EXE_NAME} PRIVATE Threads::Threads)
  target_link_libraries(${SERVER_EXE_NAME} PRIVATE ${Boost_LIBRARIES})
  target_link_libraries(${SERVER_EXE_NAME} PRIVATE ${OPENSSL_LIBRARIES})
  target_link_libraries(${SERVER_EXE_NAME} PRIVATE "-ldl")
  target_include_directories(${SERVER_EXE_NAME} PRIVATE ${MOSQUITTO_INCLUDE_DIR})
  target_link_libraries(${SERVER_EXE_NAME} PRIVATE ${MOSQUITTO_LIBRARY})

  ######
  # Setup server install and packaging

  configure_file(${CMAKE_CURRENT_SOURCE_DIR}/../../kuksa_certificates/jwt/jwt.key.pub  ${CMAKE_CURRENT_BINARY_DIR} COPYONLY)
  configure_file(${CMAKE_CURRENT_SOURCE_DIR}/../../kuksa_certificates/Server.pem  ${CMAKE_CURRENT_BINARY_DIR} COPYONLY)
  configure_file(${CMAKE_CURRENT_SOURCE_DIR}/../../kuksa_certificates/Server.key  ${CMAKE_CURRENT_BINARY_DIR} COPYONLY)
  configure_file(${CMAKE_CURRENT_SOURCE_DIR}/../../kuksa_certificates/Client.pem  ${CMAKE_CURRENT_BINARY_DIR} COPYONLY)
  configure_file(${CMAKE_CURRENT_SOURCE_DIR}/../../kuksa_certificates/Client.key  ${CMAKE_CURRENT_BINARY_DIR} COPYONLY)
  configure_file(${CMAKE_CURRENT_SOURCE_DIR}/../../kuksa_certificates/CA.pem  ${CMAKE_CURRENT_BINARY_DIR} COPYONLY)
  configure_file(${CMAKE_CURRENT_SOURCE_DIR}/../../data/vss-core/vss_release_2.0.json ${CMAKE_CURRENT_BINARY_DIR} COPYONLY)
  configure_file(${CMAKE_CURRENT_SOURCE_DIR}/../../data/vss-core/vss_release_2.1.json ${CMAKE_CURRENT_BINARY_DIR} COPYONLY)
  configure_file(${CMAKE_CURRENT_SOURCE_DIR}/../../data/vss-core/vss_release_2.2.json ${CMAKE_CURRENT_BINARY_DIR} COPYONLY)
  configure_file(${CMAKE_CURRENT_SOURCE_DIR}/../../data/vss-core/vss_release_3.0.json ${CMAKE_CURRENT_BINARY_DIR} COPYONLY)
  configure_file(${CMAKE_CURRENT_SOURCE_DIR}/../config.ini ${CMAKE_CURRENT_BINARY_DIR} COPYONLY)
  configure_file(${CMAKE_CURRENT_SOURCE_DIR}/../config_grpc_client.ini ${CMAKE_CURRENT_BINARY_DIR} COPYONLY)

  install( TARGETS ${SERVER_EXE_NAME} DESTINATION bin/${SERVER_EXE_NAME})
  install( FILES ${CMAKE_CURRENT_SOURCE_DIR}/../../kuksa_certificates/jwt/jwt.key.pub DESTINATION bin/${SERVER_EXE_NAME})
  install( FILES ${CMAKE_CURRENT_SOURCE_DIR}/../../kuksa_certificates/Server.pem DESTINATION bin/${SERVER_EXE_NAME})
  install( FILES ${CMAKE_CURRENT_SOURCE_DIR}/../../kuksa_certificates/Server.key DESTINATION bin/${SERVER_EXE_NAME})
  install( FILES ${CMAKE_CURRENT_SOURCE_DIR}/../../kuksa_certificates/Client.pem DESTINATION bin/${SERVER_EXE_NAME})
  install( FILES ${CMAKE_CURRENT_SOURCE_DIR}/../../kuksa_certificates/Client.key DESTINATION bin/${SERVER_EXE_NAME})
  install( FILES ${CMAKE_CURRENT_SOURCE_DIR}/../../kuksa_certificates/CA.pem DESTINATION bin/${SERVER_EXE_NAME})
  install( FILES ${CMAKE_CURRENT_SOURCE_DIR}/../../data/vss-core/vss_release_2.0.json DESTINATION bin/${SERVER_EXE_NAME})
  install( FILES ${CMAKE_CURRENT_SOURCE_DIR}/../../data/vss-core/vss_release_2.1.json DESTINATION bin/${SERVER_EXE_NAME})
  install( FILES ${CMAKE_CURRENT_SOURCE_DIR}/../../data/vss-core/vss_release_2.2.json DESTINATION bin/${SERVER_EXE_NAME})
  install( FILES ${CMAKE_CURRENT_SOURCE_DIR}/../../data/vss-core/vss_release_3.0.json DESTINATION bin/${SERVER_EXE_NAME})


  # CPack
  INCLUDE(InstallRequiredSystemLibraries)

  SET(CPACK_PACKAGE_DESCRIPTION_SUMMARY "${SERVER_EXE_NAME}")
  SET(CPACK_PACKAGE_VENDOR "Robert Bosch GmbH")
  # Updating project version defined at the beggining of root CMakeLists.txt file shall be used in package build
  set(CPACK_PACKAGE_VERSION_MAJOR ${PROJECT_VERSION_MAJOR})
  set(CPACK_PACKAGE_VERSION_MINOR ${PROJECT_VERSION_MINOR})
  set(CPACK_PACKAGE_VERSION_PATCH ${PROJECT_VERSION_PATCH})
  SET(CPACK_PACKAGE_DESCRIPTION_FILE "${CMAKE_CURRENT_SOURCE_DIR}/../../README.md")
  SET(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/../../LICENSE")
  SET(CPACK_PACKAGE_INSTALL_DIRECTORY "CMake ${CMake_VERSION_MAJOR}.${CMake_VERSION_MINOR}")

  SET(CPACK_GENERATOR "DEB;TGZ")
  SET(CPACK_STRIP_FILES "bin/${SERVER_EXE_NAME}")
  SET(CPACK_SOURCE_STRIP_FILES "")

  SET(CPACK_PACKAGE_EXECUTABLES "${SERVER_EXE_NAME}" "Kuksa.VAL VSS server")

  # Debian specifics
  SET(DEPENDENCY_LIBRARY_LIST
    "libc6 (>= 2.14)"
    "libgcc1 (>= 1:3.0)"
    "libssl1.0.0 (>= 1.0.1)"
    "libstdc++6 (>= 5.2)"
  )

  STRING(REPLACE ";" ", " DEPENDENCIES "${DEPENDENCY_LIBRARY_LIST}")
  SET(CPACK_DEBIAN_PACKAGE_DEPENDS "${DEPENDENCIES}")
  set(CPACK_DEBIAN_PACKAGE_MAINTAINER "John Argerus")

  INCLUDE(CPack)
endif(BUILD_EXE)
