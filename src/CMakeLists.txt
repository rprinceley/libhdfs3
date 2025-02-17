CMAKE_MINIMUM_REQUIRED(VERSION 2.8)

SET(libhdfs3_VERSION_MAJOR 2)
SET(libhdfs3_VERSION_MINOR 2)
SET(libhdfs3_VERSION_PATCH 30)
SET(libhdfs3_VERSION_STRING "${libhdfs3_VERSION_MAJOR}.${libhdfs3_VERSION_MINOR}.${libhdfs3_VERSION_PATCH}")
SET(libhdfs3_VERSION_API 1)
SET(libhdfs3_ROOT_SOURCES_DIR ${CMAKE_SOURCE_DIR}/src)
SET(libhdfs3_COMMON_SOURCES_DIR ${libhdfs3_ROOT_SOURCES_DIR}/common)

IF(ENABLE_DEBUG)
    SET(libhdfs3_VERSION_STRING "${libhdfs3_VERSION_STRING}d")
ENDIF(ENABLE_DEBUG)

CONFIGURE_FILE(platform.h.in platform.h)
CONFIGURE_FILE(doxyfile.in doxyfile)

AUTO_SOURCES(files "*.cpp" "RECURSE" "${CMAKE_CURRENT_SOURCE_DIR}")
LIST(APPEND libhdfs3_SOURCES ${files})

AUTO_SOURCES(files "*.cc" "RECURSE" "${CMAKE_CURRENT_SOURCE_DIR}")
LIST(APPEND libhdfs3_SOURCES ${files})

AUTO_SOURCES(files "*.c" "RECURSE" "${CMAKE_CURRENT_SOURCE_DIR}")
LIST(APPEND libhdfs3_SOURCES ${files})

AUTO_SOURCES(files "*.h" "RECURSE" "${CMAKE_CURRENT_SOURCE_DIR}")
LIST(APPEND libhdfs3_SOURCES ${files})

set(PROTO_INPUT_PATH ${CMAKE_SOURCE_DIR}/src/proto)
set(PROTO_OUTPUT_PATH ${CMAKE_CURRENT_BINARY_DIR}/generated)
INCLUDE_DIRECTORIES(${CMAKE_CURRENT_BINARY_DIR}/generated)

set(libhdfs3_PROTO_FILES "ClientDatanodeProtocol.proto;ClientNamenodeProtocol.proto;IpcConnectionContext.proto;ProtobufRpcEngine.proto;RpcHeader.proto;Security.proto;datatransfer.proto;hdfs.proto")
foreach(proto_file ${libhdfs3_PROTO_FILES})
    get_filename_component(bcos_proto_abs "${PROTO_INPUT_PATH}" ABSOLUTE)
    set(proto_file_abs ${bcos_proto_abs}/${proto_file})
    get_filename_component(rel_dir ${proto_file} DIRECTORY)
    get_filename_component(basename ${proto_file} NAME_WE)
    set(generated_files ${PROTO_OUTPUT_PATH}/${basename}.pb.cc)

    list(APPEND PROTO_SRCS ${generated_files})
    message("Command: protoc --cpp_out ${PROTO_OUTPUT_PATH} -I ${PROTO_INPUT_PATH} ${proto_file}")
    add_custom_command(
        OUTPUT ${generated_files}
        COMMAND protobuf::protoc --cpp_out ${PROTO_OUTPUT_PATH} -I ${PROTO_INPUT_PATH} ${proto_file}
        COMMENT "Generating ${generated_files} from ${proto_file_abs}"
        VERBATIM
    )
endforeach()

SET(HEADER 
    client/BlockLocation.h
    client/DirectoryIterator.h
    client/FileStatus.h
    client/FileSystem.h
    client/FileSystemStats.h
    client/hdfs.h
    client/InputStream.h
    client/OutputStream.h
    client/Permission.h
    common/Exception.h
    common/XmlConfig.h)

ADD_LIBRARY(libhdfs3-static STATIC ${libhdfs3_SOURCES} ${libhdfs3_PROTO_SOURCES} ${PROTO_SRCS})
ADD_LIBRARY(libhdfs3-shared SHARED ${libhdfs3_SOURCES} ${libhdfs3_PROTO_SOURCES} ${PROTO_SRCS})

ADD_CUSTOM_COMMAND(
    TARGET libhdfs3-shared libhdfs3-static
    PRE_BUILD
    COMMAND echo ${libhdfs3_VERSION_STRING} > version
    WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
)

TARGET_LINK_LIBRARIES(libhdfs3-static pthread)
TARGET_LINK_LIBRARIES(libhdfs3-shared pthread)
       
IF(NEED_BOOST)
    INCLUDE_DIRECTORIES(${Boost_INCLUDE_DIR})
    TARGET_LINK_LIBRARIES(libhdfs3-static boost_thread)
    TARGET_LINK_LIBRARIES(libhdfs3-static boost_chrono)
    TARGET_LINK_LIBRARIES(libhdfs3-static boost_system)
    TARGET_LINK_LIBRARIES(libhdfs3-static boost_atomic)
    TARGET_LINK_LIBRARIES(libhdfs3-static boost_iostreams)
    TARGET_LINK_LIBRARIES(libhdfs3-shared boost_thread)
    TARGET_LINK_LIBRARIES(libhdfs3-shared boost_chrono)
    TARGET_LINK_LIBRARIES(libhdfs3-shared boost_system)
    TARGET_LINK_LIBRARIES(libhdfs3-shared boost_atomic)
    TARGET_LINK_LIBRARIES(libhdfs3-shared boost_iostreams)
ENDIF(NEED_BOOST)

IF(NEED_GCCEH)
    TARGET_LINK_LIBRARIES(libhdfs3-static gcc_eh)
    TARGET_LINK_LIBRARIES(libhdfs3-shared gcc_eh)
ENDIF(NEED_GCCEH)

IF(OS_LINUX)
    TARGET_LINK_LIBRARIES(libhdfs3-static PkgConfig::uuid)
    TARGET_LINK_LIBRARIES(libhdfs3-shared PRIVATE PkgConfig::uuid)
ENDIF(OS_LINUX)

INCLUDE_DIRECTORIES(${libhdfs3_ROOT_SOURCES_DIR})
INCLUDE_DIRECTORIES(${libhdfs3_COMMON_SOURCES_DIR})
INCLUDE_DIRECTORIES(${CMAKE_CURRENT_BINARY_DIR})

IF (WITH_KERBEROS)
INCLUDE_DIRECTORIES(${KERBEROS_INCLUDE_DIRS})
ENDIF()

INCLUDE_DIRECTORIES(${GSASL_INCLUDE_DIR})
INCLUDE_DIRECTORIES(${CMAKE_SOURCE_DIR}/mock)

TARGET_LINK_LIBRARIES(libhdfs3-static PRIVATE protobuf::libprotobuf)
TARGET_LINK_LIBRARIES(libhdfs3-static libxml2::libxml2)

IF (WITH_KERBEROS)
TARGET_LINK_LIBRARIES(libhdfs3-static ${KERBEROS_LIBRARIES})
ENDIF()

IF (APPLE)
TARGET_LINK_LIBRARIES(libhdfs3-static ${GSASL_LIBRARIES})
ELSE()
TARGET_LINK_LIBRARIES(libhdfs3-static ${GSASL_STATIC_LIBRARIES})
ENDIF()

TARGET_LINK_LIBRARIES(libhdfs3-shared PRIVATE protobuf::libprotobuf)
TARGET_LINK_LIBRARIES(libhdfs3-shared libxml2::libxml2)

IF (WITH_KERBEROS)
TARGET_LINK_LIBRARIES(libhdfs3-shared ${KERBEROS_LIBRARIES})
ENDIF()

TARGET_LINK_LIBRARIES(libhdfs3-shared ${GSASL_LIBRARIES})

SET_TARGET_PROPERTIES(libhdfs3-static PROPERTIES OUTPUT_NAME "hdfs3")
SET_TARGET_PROPERTIES(libhdfs3-shared PROPERTIES OUTPUT_NAME "hdfs3")

IF(NEED_BOOST)
    SET_TARGET_PROPERTIES(libhdfs3-shared libhdfs3-static PROPERTIES LINK_FLAGS "-L${Boost_LIBRARY_DIRS}")
ENDIF(NEED_BOOST)

SET_TARGET_PROPERTIES(libhdfs3-shared PROPERTIES 
    VERSION ${libhdfs3_VERSION_MAJOR}.${libhdfs3_VERSION_MINOR}.${libhdfs3_VERSION_PATCH} 
    SOVERSION ${libhdfs3_VERSION_API})

include(InstallConfig)
install(
    TARGETS libhdfs3-static libhdfs3-shared
    EXPORT "${TARGETS_EXPORT_NAME}"
    LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}"
    ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}"
    RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}"
    INCLUDES DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
)
INSTALL(FILES ${HEADER} DESTINATION include/hdfs)
INSTALL(FILES libhdfs3.pc DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig)

SET(libhdfs3_SOURCES ${libhdfs3_SOURCES} PARENT_SCOPE)
SET(libhdfs3_PLATFORM_HEADER_DIR ${CMAKE_CURRENT_BINARY_DIR} PARENT_SCOPE)
SET(libhdfs3_ROOT_SOURCES_DIR ${libhdfs3_ROOT_SOURCES_DIR} PARENT_SCOPE)
SET(libhdfs3_COMMON_SOURCES_DIR ${libhdfs3_COMMON_SOURCES_DIR} PARENT_SCOPE)
