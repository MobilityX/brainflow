
set (GANGLION_SOURCE_LIB
    callbacks.cpp
    cmd_def.cpp
    helpers.cpp
    main.cpp
    stubs.cpp
    uart.cpp
    ${CMAKE_HOME_DIRECTORY}/src/utils/timestamp.cpp
)
add_library (GanglionLib SHARED ${GANGLION_SOURCE_LIB})
target_include_directories (GanglionLib PUBLIC
    $<BUILD_INTERFACE:${CMAKE_HOME_DIRECTORY}/src/utils/inc>
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/inc>
)
set_property (TARGET GanglionLib PROPERTY POSITION_INDEPENDENT_CODE ON)

if (CMAKE_SIZEOF_VOID_P EQUAL 8)
    set_property (TARGET GanglionLib PROPERTY OUTPUT_NAME GanglionLib64)
else (CMAKE_SIZEOF_VOID_P EQUAL 8)
    set_property (TARGET GanglionLib PROPERTY OUTPUT_NAME GanglionLib32)
endif (CMAKE_SIZEOF_VOID_P EQUAL 8)