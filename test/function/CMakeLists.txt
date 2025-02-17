CMAKE_MINIMUM_REQUIRED(VERSION 2.8)

AUTO_SOURCES(function_SOURCES "*.cpp" "RECURSE" "${CMAKE_CURRENT_SOURCE_DIR}")

INCLUDE_DIRECTORIES(${gmock_INCLUDE_DIR} ${gtest_INCLUDE_DIR} ${libhdfs3_ROOT_SOURCES_DIR})

IF(NEED_BOOST)
    INCLUDE_DIRECTORIES(${Boost_INCLUDE_DIR})
ENDIF(NEED_BOOST)

INCLUDE_DIRECTORIES(${libhdfs3_ROOT_SOURCES_DIR})
INCLUDE_DIRECTORIES(${libhdfs3_COMMON_SOURCES_DIR})
INCLUDE_DIRECTORIES(${CMAKE_CURRENT_BINARY_DIR})
INCLUDE_DIRECTORIES(${libhdfs3_PLATFORM_HEADER_DIR})

IF (WITH_KERBEROS)
INCLUDE_DIRECTORIES(${KERBEROS_INCLUDE_DIRS})
ENDIF()

INCLUDE_DIRECTORIES(${GSASL_INCLUDE_DIR})
INCLUDE_DIRECTORIES(${CMAKE_SOURCE_DIR}/mock)

IF(ENABLE_DEBUG STREQUAL ON)
    SET(libhdfs3_SOURCES ${libhdfs3_SOURCES} ${libhdfs3_MOCK_SOURCES})
ENDIF(ENABLE_DEBUG STREQUAL ON)

IF(NOT HDFS_SUPERUSER)
    SET(HDFS_SUPERUSER $ENV{USER})
ENDIF(NOT HDFS_SUPERUSER)

ADD_DEFINITIONS(-DHDFS_SUPERUSER="${HDFS_SUPERUSER}")
ADD_DEFINITIONS(-DUSER="$ENV{USER}")

ADD_EXECUTABLE(function EXCLUDE_FROM_ALL
    ${gtest_SOURCES}
    ${gmock_SOURCES}
    ${libhdfs3_SOURCES} 
    ${libhdfs3_PROTO_SOURCES} 
    ${libhdfs3_PROTO_HEADERS}
    ${function_SOURCES}
)

TARGET_LINK_LIBRARIES(function pthread)

IF(NEED_BOOST)
    INCLUDE_DIRECTORIES(${Boost_INCLUDE_DIR})
    SET(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -L${Boost_LIBRARY_DIRS}")
    TARGET_LINK_LIBRARIES(function boost_thread)
    TARGET_LINK_LIBRARIES(function boost_chrono)
    TARGET_LINK_LIBRARIES(function boost_system)
    TARGET_LINK_LIBRARIES(function boost_atomic)
    TARGET_LINK_LIBRARIES(function boost_iostreams)
ENDIF(NEED_BOOST)

IF(NEED_GCCEH)
    TARGET_LINK_LIBRARIES(function gcc_eh)
ENDIF(NEED_GCCEH)

IF(OS_LINUX)
    TARGET_LINK_LIBRARIES(function PkgConfig::uuid)
ENDIF(OS_LINUX)

TARGET_LINK_LIBRARIES(function protobuf::libprotobuf)
TARGET_LINK_LIBRARIES(function libxml2::libxml2)

IF (WITH_KERBEROS)
TARGET_LINK_LIBRARIES(function ${KERBEROS_LIBRARIES})
ENDIF()

TARGET_LINK_LIBRARIES(function ${GSASL_LIBRARIES})

SET(function_SOURCES ${function_SOURCES} PARENT_SCOPE)

