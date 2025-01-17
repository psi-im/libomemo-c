if(BUILD_SHARED_LIBS AND NOT MSVC)
    find_library(M_LIB m)
endif()
if(BUILD_WITH_PROTOBUF STREQUAL bundled)
    include(${PROJECT_SOURCE_DIR}/CMakeModules/protobuf_c_bundled.cmake)
else()
    find_package(Protobuf_C REQUIRED)
endif()


set(protobuf_SRCS
    LocalStorageProtocol.pb-c.c
    WhisperTextProtocol.pb-c.c
    OMEMO.pb-c.c
)

set(omemo_SRCS
    vpool.c
    vpool.h
    signal_protocol.c
    signal_protocol.h
    signal_protocol_types.h
    signal_protocol_internal.h
    curve.c
    curve.h
    hkdf.c
    hkdf.h
    ratchet.c
    ratchet.h
    protocol.c
    protocol.h
    session_state.c
    session_state.h
    session_record.c
    session_record.h
    session_pre_key.c
    session_pre_key.h
    session_builder.c
    session_builder.h
    session_builder_internal.h
    session_cipher.c
    session_cipher.h
    key_helper.c
    key_helper.h
    sender_key.c
    sender_key.h
    sender_key_state.c
    sender_key_state.h
    sender_key_record.c
    sender_key_record.h
)

add_subdirectory(curve25519)

add_library(${PROJECT_NAME}
    ${protobuf_SRCS}
    ${omemo_SRCS}
    $<TARGET_OBJECTS:curve25519>
)

if(BUILD_WITH_PROTOBUF STREQUAL bundled)
    add_dependencies(${PROJECT_NAME} ProtobufCProject)
    # linking to Protobuf_C::Protobuf_C generates invald config.cmake. So use files instead
    target_link_libraries(${PROJECT_NAME} PUBLIC
        "$<BUILD_INTERFACE:${Protobuf_C_LIBRARY}>"
        "$<INSTALL_INTERFACE:\${_IMPORT_PREFIX}/${CMAKE_INSTALL_LIBDIR}/${Protobuf_C_LIBRARY_NAME}>")
    target_include_directories(${PROJECT_NAME} PRIVATE ${Protobuf_C_INCLUDE_DIR})
elseif(BUILD_WITH_PROTOBUF STREQUAL static)
    target_link_libraries(${PROJECT_NAME} PUBLIC Protobuf_C::Protobuf_C)
else()
    target_link_libraries(${PROJECT_NAME} PRIVATE Protobuf_C::Protobuf_C)
endif()

if(BUILD_SHARED_LIBS)
    if(NOT MSVC)
        target_link_libraries(${PROJECT_NAME} PRIVATE ${M_LIB})
    endif()
    set_target_properties(${PROJECT_NAME} PROPERTIES
        VERSION ${OMEMO_C_VERSION}
        SOVERSION ${OMEMO_C_VERSION_MAJOR}
    )
endif()

target_include_directories(${PROJECT_NAME} PRIVATE
    curve25519/ed25519/nacl_includes
    curve25519/ed25519/additions
    curve25519/ed25519/sha512
    curve25519/ed25519
    curve25519
)

target_include_directories(${PROJECT_NAME} PUBLIC
   "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>"
   "$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}/omemo>"
)

install(
    FILES
    signal_protocol.h
    signal_protocol_types.h
    curve.h
    hkdf.h
    ratchet.h
    protocol.h
    session_state.h
    session_record.h
    session_pre_key.h
    session_builder.h
    session_cipher.h
    key_helper.h
    sender_key.h
    sender_key_state.h
    sender_key_record.h
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/omemo
)

install(TARGETS ${PROJECT_NAME} EXPORT ${PROJECT_NAME}-targets
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
)

if(GENERATE_PKGCONFIG)
    # see https://github.com/jupp0r/prometheus-cpp/issues/587
    if(IS_ABSOLUTE "${CMAKE_INSTALL_INCLUDEDIR}")
        set(OMEMO_C_PKGCONFIG_INCLUDEDIR "${CMAKE_INSTALL_INCLUDEDIR}")
    else()
        set(OMEMO_C_PKGCONFIG_INCLUDEDIR "\${prefix}/${CMAKE_INSTALL_INCLUDEDIR}")
    endif()

    if(IS_ABSOLUTE "${CMAKE_INSTALL_LIBDIR}")
        set(OMEMO_C_PKGCONFIG_LIBDIR "${CMAKE_INSTALL_LIBDIR}")
    else()
        set(OMEMO_C_PKGCONFIG_LIBDIR "\${exec_prefix}/${CMAKE_INSTALL_LIBDIR}")
    endif()

    configure_file( ${CMAKE_CURRENT_SOURCE_DIR}/lib${PROJECT_NAME}.pc.in
        ${CMAKE_CURRENT_BINARY_DIR}/lib${PROJECT_NAME}.pc @ONLY)

    install(FILES ${CMAKE_CURRENT_BINARY_DIR}/lib${PROJECT_NAME}.pc DESTINATION "${INSTALL_PKGCONFIG_DIR}")
endif()


export(EXPORT ${PROJECT_NAME}-targets
  FILE "${CMAKE_CURRENT_BINARY_DIR}/cmake/${PROJECT_NAME}/${PROJECT_NAME}-targets.cmake"
  NAMESPACE ${PROJECT_NAME}::
)


set(ConfigPackageLocation ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME})
install(
    EXPORT ${PROJECT_NAME}-targets
    NAMESPACE ${PROJECT_NAME}::
    FILE ${PROJECT_NAME}-targets.cmake
    DESTINATION "${ConfigPackageLocation}"
)

include(CMakePackageConfigHelpers)
configure_package_config_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/${PROJECT_NAME}-config.cmake.in"
    ${PROJECT_NAME}-config.cmake
    INSTALL_DESTINATION "${ConfigPackageLocation}"
    NO_CHECK_REQUIRED_COMPONENTS_MACRO
    PATH_VARS CMAKE_INSTALL_INCLUDEDIR
)
install(
  FILES "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-config.cmake"
  DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}"
)
