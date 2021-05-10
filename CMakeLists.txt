cmake_minimum_required (VERSION 3.13)
project (brainflow)

set (CMAKE_CXX_STANDARD 11)
set (CMAKE_VERBOSE_MAKEFILE ON)
set (BRAINFLOW_VERSION 3.7.3)

# dont export sumbols on unix by default
#set (CMAKE_CXX_VISIBILITY_PRESET hidden)
#set (CMAKE_C_VISIBILITY_PRESET hidden)

set (CMAKE_POSITION_INDEPENDENT_CODE ON)

option(DISABLE_AS_DEFAULT "Disable hardware support by default" OFF)
if (DISABLE_AS_DEFAULT)
    SET(ENABLE_CYTHON OFF)
    SET(ENABLE_GANGLION OFF)
    SET(ENABLE_GALEA OFF)
    SET(ENABLE_BRAINBIT OFF)
    SET(ENABLE_BRAINBIT_BLED OFF)
    SET(ENABLE_UNICORN OFF)
    SET(ENABLE_CALLIBRI OFF)
    SET(ENABLE_FASCIA OFF)
    SET(ENABLE_NOTION OFF)
    SET(ENABLE_IRONBCI OFF)
    SET(ENABLE_GFORCE_PRO OFF)
    SET(ENABLE_GFORCE_DUAL OFF)
    SET(ENABLE_FREEEEG32 OFF)
endif (DISABLE_AS_DEFAULT)

option(ENABLE_CYTHON "ENABLE_CYTHON" ON)
option(ENABLE_GANGLION "ENABLE_GANGLION" ON)
option(ENABLE_GALEA "ENABLE_GALEA" ON)
option(ENABLE_BRAINBIT "ENABLE_BRAINBIT" OFF)
option(ENABLE_BRAINBIT_BLED "ENABLE_BRAINBIT_BLED" ON)
option(ENABLE_UNICORN "ENABLE_UNICORN" OFF)
option(ENABLE_CALLIBRI "ENABLE_CALLIBRI" OFF)
option(ENABLE_FASCIA "ENABLE_FASCIA" ON)
option(ENABLE_NOTION "ENABLE_NOTION" ON)
option(ENABLE_IRONBCI "ENABLE_IRONBCI" ON)
option(ENABLE_GFORCE_PRO "ENABLE_GFORCE_PRO" OFF)
option(ENABLE_GFORCE_DUAL "ENABLE_GFORCE_DUAL" OFF)
option(ENABLE_FREEEEG32 "ENABLE_FREEEEG32" ON)

option(USE_LIBFTDI "USE_LIBFTDI" OFF)
option(USE_OPENMP "USE_OPENMP" OFF)
option(WARNINGS_AS_ERRORS "WARNINGS_AS_ERRORS" OFF)
option(BUILD_OYMOTION_SDK "BUILD_OYMOTION_SDK" OFF)

macro (configure_msvc_runtime)
    if (MSVC)
        add_definitions(-D_CRT_SECURE_NO_WARNINGS -D_SILENCE_TR1_NAMESPACE_DEPRECATION_WARNING -D_CRT_NONSTDC_NO_DEPRECATE)
        # Default to statically-linked runtime.
        if ("${MSVC_RUNTIME}" STREQUAL "")
            set (MSVC_RUNTIME "static")
        endif ()
        # Set compiler options.
        set (variables
            CMAKE_C_FLAGS_DEBUG
            CMAKE_C_FLAGS_MINSIZEREL
            CMAKE_C_FLAGS_RELEASE
            CMAKE_C_FLAGS_RELWITHDEBINFO
            CMAKE_CXX_FLAGS_DEBUG
            CMAKE_CXX_FLAGS_MINSIZEREL
            CMAKE_CXX_FLAGS_RELEASE
            CMAKE_CXX_FLAGS_RELWITHDEBINFO
        )
        if (${MSVC_RUNTIME} STREQUAL "static")
            message(STATUS
                "MSVC -> forcing use of statically-linked runtime."
            )
            foreach (variable ${variables})
                if (${variable} MATCHES "/MD")
                    string (REGEX REPLACE "/MD" "/MT" ${variable} "${${variable}}")
                endif ()
            endforeach ()
        else ()
            message (STATUS
                "MSVC -> forcing use of dynamically-linked runtime."
            )
            foreach (variable ${variables})
                if (${variable} MATCHES "/MT")
                    string (REGEX REPLACE "/MT" "/MD" ${variable} "${${variable}}")
                endif ()
            endforeach ()
        endif ()
    endif ()
endmacro ()

# link msvc runtime statically
configure_msvc_runtime ()

# warnings as errors but with default(low) level
if (WARNINGS_AS_ERRORS)
    if (MSVC)
        add_compile_options(/WX)
    else()
        add_compile_options(-Werror -Wno-varargs)
    endif()
endif (WARNINGS_AS_ERRORS)

# add_subfolder doesnt work on linux if you need to istall packages, add it in top level file
SET (DSPFILTERS "DSPFilters")
aux_source_directory (${CMAKE_HOME_DIRECTORY}/third_party/DSPFilters/source DSPFILTERS_SOURCE_LIB)
add_library (${DSPFILTERS} STATIC ${DSPFILTERS_SOURCE_LIB})
target_include_directories (${DSPFILTERS} PUBLIC $<BUILD_INTERFACE:${CMAKE_HOME_DIRECTORY}/third_party/DSPFilters/include>)
set_property (TARGET ${DSPFILTERS} PROPERTY POSITION_INDEPENDENT_CODE ON)
# there are asserts, I wanna disable them even for debug builds
target_compile_definitions(${DSPFILTERS} PRIVATE -DNDEBUG)

SET (WAVELIB "WaveLib")
aux_source_directory (${CMAKE_HOME_DIRECTORY}/third_party/wavelib/src WAVELIB_SRC)
add_library (${WAVELIB} STATIC ${WAVELIB_SRC})
target_include_directories (${WAVELIB} PUBLIC $<BUILD_INTERFACE:${CMAKE_HOME_DIRECTORY}/third_party/wavelib/header>)
set_property (TARGET ${WAVELIB} PROPERTY POSITION_INDEPENDENT_CODE ON)


set (BOARD_CONTROLLER_SRC
    ${CMAKE_HOME_DIRECTORY}/src/utils/timestamp.cpp
    ${CMAKE_HOME_DIRECTORY}/src/utils/data_buffer.cpp
    ${CMAKE_HOME_DIRECTORY}/src/utils/os_serial.cpp
    ${CMAKE_HOME_DIRECTORY}/src/utils/os_serial_ioctl.cpp
    ${CMAKE_HOME_DIRECTORY}/src/utils/serial.cpp
    ${CMAKE_HOME_DIRECTORY}/src/utils/libftdi_serial.cpp
    ${CMAKE_HOME_DIRECTORY}/src/utils/socket_client_tcp.cpp
    ${CMAKE_HOME_DIRECTORY}/src/utils/socket_client_udp.cpp
    ${CMAKE_HOME_DIRECTORY}/src/utils/socket_server_tcp.cpp
    ${CMAKE_HOME_DIRECTORY}/src/utils/socket_server_udp.cpp
    ${CMAKE_HOME_DIRECTORY}/src/utils/multicast_client.cpp
    ${CMAKE_HOME_DIRECTORY}/src/utils/multicast_server.cpp
    ${CMAKE_HOME_DIRECTORY}/src/utils/broadcast_client.cpp
    ${CMAKE_HOME_DIRECTORY}/src/board_controller/board_controller.cpp
    ${CMAKE_HOME_DIRECTORY}/src/board_controller/board_info_getter.cpp
    ${CMAKE_HOME_DIRECTORY}/src/board_controller/board.cpp
    ${CMAKE_HOME_DIRECTORY}/src/board_controller/brainflow_boards.cpp
    ${CMAKE_HOME_DIRECTORY}/src/board_controller/streaming_board.cpp
    ${CMAKE_HOME_DIRECTORY}/src/board_controller/synthetic_board.cpp
    ${CMAKE_HOME_DIRECTORY}/src/board_controller/playback_file_board.cpp
    ${CMAKE_HOME_DIRECTORY}/src/board_controller/file_streamer.cpp
    ${CMAKE_HOME_DIRECTORY}/src/board_controller/multicast_streamer.cpp
)

set (BOARD_CONTROLLER_HDR
    src/board_controller/inc/board_controller.h
    src/board_controller/inc/board_info_getter.h
    src/board_controller/inc/brainflow_input_params.h
    src/utils/inc/brainflow_constants.h
    src/utils/inc/shared_export.h
)

set (DATA_HANDLER_SRC
    ${CMAKE_HOME_DIRECTORY}/src/data_handler/data_handler.cpp
)

set (DATA_HANDLER_HDR
    src/data_handler/inc/data_handler.h
    src/utils/inc/brainflow_constants.h
    src/utils/inc/shared_export.h
)

set (ML_MODULE_SRC
    ${CMAKE_HOME_DIRECTORY}/src/ml/ml_module.cpp
    ${CMAKE_HOME_DIRECTORY}/src/ml/concentration_regression_classifier.cpp
    ${CMAKE_HOME_DIRECTORY}/src/ml/base_classifier.cpp
    ${CMAKE_HOME_DIRECTORY}/third_party/libsvm/svm.cpp
    ${CMAKE_HOME_DIRECTORY}/src/ml/concentration_knn_classifier.cpp
    ${CMAKE_HOME_DIRECTORY}/src/ml/concentration_svm_classifier.cpp
    ${CMAKE_HOME_DIRECTORY}/src/ml/concentration_lda_classifier.cpp
    ${CMAKE_HOME_DIRECTORY}/src/ml/generated/focus_dataset.cpp
    ${CMAKE_HOME_DIRECTORY}/src/ml/generated/lda_model.cpp
    ${CMAKE_HOME_DIRECTORY}/src/ml/generated/regression_model.cpp
)

set (ML_MODULE_HDR
    src/ml/inc/ml_module.h
    src/ml/inc/brainflow_model_params.h
    src/utils/inc/brainflow_constants.h
    src/utils/inc/shared_export.h
)


add_library (
    BoardController SHARED
    ${BOARD_CONTROLLER_SRC}
)

add_library (
    DataHandler SHARED
    ${DATA_HANDLER_SRC}
)

add_library (
    MLModule SHARED
    ${ML_MODULE_SRC}
)

configure_file (
    ${CMAKE_HOME_DIRECTORY}/src/board_controller/inc/board_controller_config.h.in
    ${CMAKE_BINARY_DIR}/src/board_controller/inc/board_controller_config.h
)

set_target_properties(BoardController PROPERTIES PUBLIC_HEADER "${BOARD_CONTROLLER_HDR}")

target_include_directories (
    BoardController PRIVATE
    ${CMAKE_BINARY_DIR}/src/board_controller/inc
    ${CMAKE_HOME_DIRECTORY}/third_party/
    ${CMAKE_HOME_DIRECTORY}/third_party/json
    ${CMAKE_HOME_DIRECTORY}/third_party/http
    ${CMAKE_HOME_DIRECTORY}/third_party/oscpp/include
    ${CMAKE_HOME_DIRECTORY}/src/utils/inc
    ${CMAKE_HOME_DIRECTORY}/src/board_controller/inc
    ${CMAKE_HOME_DIRECTORY}/src/board_controller/openbci/inc
    ${CMAKE_HOME_DIRECTORY}/src/board_controller/oymotion/inc
    ${CMAKE_HOME_DIRECTORY}/src/board_controller/gtec/inc
    ${CMAKE_HOME_DIRECTORY}/src/board_controller/neuromd/inc
    ${CMAKE_HOME_DIRECTORY}/src/board_controller/openbci/ganglion_bglib/inc
    ${CMAKE_HOME_DIRECTORY}/src/board_controller/neuromd/brainbit_bglib/inc
    ${CMAKE_HOME_DIRECTORY}/src/board_controller/mit/inc
    ${CMAKE_HOME_DIRECTORY}/src/board_controller/neurosity/inc
    ${CMAKE_HOME_DIRECTORY}/src/board_controller/ironbci/inc
    ${CMAKE_HOME_DIRECTORY}/src/board_controller/freeeeg32/inc
)

# disable min max defines in windows.h
target_compile_definitions(BoardController PRIVATE -DNOMINMAX)


set_target_properties(DataHandler PROPERTIES PUBLIC_HEADER "${DATA_HANDLER_HDR}")

target_include_directories (
    DataHandler PRIVATE
    ${CMAKE_HOME_DIRECTORY}/third_party/
    ${CMAKE_HOME_DIRECTORY}/src/utils/inc
    ${CMAKE_HOME_DIRECTORY}/src/board_controller/inc
    ${CMAKE_HOME_DIRECTORY}/src/data_handler/inc
    ${CMAKE_HOME_DIRECTORY}/third_party/DSPFilters/include
    ${CMAKE_HOME_DIRECTORY}/third_party/wavelib/header
    ${CMAKE_HOME_DIRECTORY}/third_party/fft/src
)

set_target_properties(MLModule PROPERTIES PUBLIC_HEADER "${ML_MODULE_HDR}")

target_include_directories (
    MLModule PRIVATE
    ${CMAKE_HOME_DIRECTORY}/third_party/
    ${CMAKE_HOME_DIRECTORY}/src/utils/inc
    ${CMAKE_HOME_DIRECTORY}/src/ml/inc
    ${CMAKE_HOME_DIRECTORY}/third_party/libsvm
    ${CMAKE_HOME_DIRECTORY}/third_party/json
    ${CMAKE_HOME_DIRECTORY}/third_party/kdtree
)

if (ENABLE_CYTHON)
    target_sources(
        BoardController PRIVATE
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/openbci/cyton_wifi.cpp
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/openbci/cyton_daisy_wifi.cpp
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/openbci/cyton.cpp
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/openbci/cyton_daisy.cpp
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/openbci/openbci_serial_board.cpp
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/openbci/openbci_wifi_shield_board.cpp
    )
endif(ENABLE_CYTHON)

if (ENABLE_GANGLION)
    target_sources(
        BoardController PRIVATE
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/openbci/openbci_serial_board.cpp
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/openbci/openbci_wifi_shield_board.cpp
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/openbci/ganglion_wifi.cpp
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/openbci/ganglion.cpp
    )
    add_subdirectory(src/board_controller/openbci/ganglion_bglib)
endif(ENABLE_GANGLION)


if (ENABLE_GALEA)
    target_sources(
        BoardController PRIVATE
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/openbci/galea_serial.cpp
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/openbci/galea.cpp
    )
endif(ENABLE_GALEA)


if (ENABLE_BRAINBIT)
    target_sources(
        BoardController PRIVATE
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/neuromd/neuromd_board.cpp
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/neuromd/brainbit.cpp
    )
    target_include_directories (
        BoardController PRIVATE
        ${CMAKE_HOME_DIRECTORY}/third_party/neurosdk/inc/types
    )
endif(ENABLE_BRAINBIT)


if (ENABLE_BRAINBIT_BLED)
    target_sources(
        BoardController PRIVATE
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/neuromd/brainbit_bled.cpp
    )
    add_subdirectory(src/board_controller/neuromd/brainbit_bglib)
endif(ENABLE_BRAINBIT_BLED)


if (ENABLE_UNICORN)
    target_sources(
        BoardController PRIVATE
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/gtec/unicorn_board.cpp
    )
    target_include_directories (
        BoardController PRIVATE
        ${CMAKE_HOME_DIRECTORY}/third_party/unicorn/inc
    )
        
endif(ENABLE_UNICORN)


if (ENABLE_CALLIBRI)
    target_sources(
        BoardController PRIVATE
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/neuromd/neuromd_board.cpp
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/neuromd/callibri.cpp
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/neuromd/callibri_eeg.cpp
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/neuromd/callibri_ecg.cpp
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/neuromd/callibri_emg.cpp
    )
    target_include_directories (
        BoardController PRIVATE
        ${CMAKE_HOME_DIRECTORY}/third_party/neurosdk/inc/types
    )
endif(ENABLE_CALLIBRI)


if (ENABLE_FASCIA)
    target_sources(
        BoardController PRIVATE
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/mit/fascia.cpp
    )
endif(ENABLE_FASCIA)


if (ENABLE_NOTION)
    target_sources(
        BoardController PRIVATE
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/neurosity/notion_osc.cpp
    )
endif(ENABLE_NOTION)


if (ENABLE_IRONBCI)
    target_sources(
        BoardController PRIVATE
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/ironbci/ironbci.cpp
    )
endif (ENABLE_IRONBCI)


if (ENABLE_GFORCE_PRO)
    target_sources(
        BoardController PRIVATE
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/oymotion/gforce_pro.cpp
    )
    target_include_directories (
        BoardController PRIVATE
        ${CMAKE_HOME_DIRECTORY}/third_party/gForceSDKCXX/src/inc
    )
endif (ENABLE_GFORCE_PRO)


if (ENABLE_GFORCE_DUAL)
    target_sources(
        BoardController PRIVATE
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/oymotion/gforce_dual.cpp
    )
    target_include_directories (
        BoardController PRIVATE
        ${CMAKE_HOME_DIRECTORY}/third_party/gForceSDKCXX/src/inc
    )
endif (ENABLE_GFORCE_DUAL)


if (ENABLE_FREEEEG32)
    target_sources(
        BoardController PRIVATE
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/freeeeg32/freeeeg32.cpp
    )
endif (ENABLE_FREEEEG32)


if (BUILD_OYMOTION_SDK)
    add_subdirectory (${CMAKE_HOME_DIRECTORY}/third_party/gForceSDKCXX)
endif (BUILD_OYMOTION_SDK)

if (USE_LIBFTDI)
    find_package (LibFTDI1 NO_MODULE)
    if (LibFTDI1_FOUND)
        # this line could set up the directories if included before targets are defined
        #include ( ${LIBFTDI_USE_FILE} )

        # this works around a bug in some installations of libftdi++ 1.5
        # http://developer.intra2net.com/git/?p=libftdi;a=commit;h=cdb28383402d248dbc6062f4391b038375c52385
        set (LIBFTDI_INCLUDE_DIRS ${LIBFTDI_INCLUDE_DIRS} ${LIBFTDI_INCLUDE_DIR}/../libftdi1)

        target_compile_definitions (BoardController PRIVATE ${LIBFTDI_DEFINITIONS} "USE_LIBFTDI")
        target_include_directories (BoardController PRIVATE ${LIBFTDI_INCLUDE_DIRS})
        target_link_directories (BoardController PRIVATE ${LIBFTDI_LIBRARY_DIRS})
        target_link_libraries (BoardController PRIVATE ${LIBFTDI_LIBRARIES})
    else (LibFTDI1_FOUND)
        message (FATAL_ERROR "USE_LIBFTDI set but LibFTDI not found.")
    endif (LibFTDI1_FOUND)
endif (USE_LIBFTDI)

if (USE_OPENMP)
    find_package (OpenMP)
    if (OpenMP_CXX_FOUND)
        target_link_libraries (DataHandler PRIVATE OpenMP::OpenMP_CXX)
        target_link_libraries (MLModule PRIVATE OpenMP::OpenMP_CXX)
    else (OpenMP_CXX_FOUND)
        message (FATAL_ERROR "ENABLE_OPENMP set but no OpenMP found.")
    endif (OpenMP_CXX_FOUND)
endif (USE_OPENMP)

# dont link pthread for Android
if (UNIX AND NOT ANDROID)
    target_link_libraries (BoardController PRIVATE pthread dl)
    target_link_libraries (MLModule PRIVATE pthread dl)
    target_link_libraries (DataHandler PRIVATE ${DSPFILTERS} ${WAVELIB} pthread dl)
else (UNIX AND NOT ANDROID)
    target_link_libraries (DataHandler PRIVATE ${DSPFILTERS} ${WAVELIB})
endif (UNIX AND NOT ANDROID)
# link android logging library
if (ANDROID)
    find_library (log-lib log)
    target_link_libraries (MLModule PRIVATE log)
    target_link_libraries (BoardController PRIVATE log)
    target_link_libraries (DataHandler PRIVATE  log)
endif (ANDROID)


set (BRAINFLOW_HDR
    cpp-package/src/inc/data_filter.h
    cpp-package/src/inc/board_shim.h
    cpp-package/src/inc/ml_model.h
    src/utils/inc/brainflow_array.h
    src/utils/inc/brainflow_exception.h
)

add_library (
    Brainflow STATIC
    ${CMAKE_HOME_DIRECTORY}/cpp-package/src/board_shim.cpp
    ${CMAKE_HOME_DIRECTORY}/cpp-package/src/ml_model.cpp
    ${CMAKE_HOME_DIRECTORY}/cpp-package/src/data_filter.cpp
)

set_target_properties(Brainflow PROPERTIES PUBLIC_HEADER "${BRAINFLOW_HDR}")

target_include_directories (
    Brainflow PRIVATE
    ${CMAKE_HOME_DIRECTORY}/src/board_controller/inc
    ${CMAKE_HOME_DIRECTORY}/src/utils/inc
    ${CMAKE_HOME_DIRECTORY}/src/data_handler/inc
    ${CMAKE_HOME_DIRECTORY}/cpp-package/src/inc
    ${CMAKE_HOME_DIRECTORY}/third_party/json
    ${CMAKE_HOME_DIRECTORY}/src/ml/inc
)


target_link_libraries (Brainflow PRIVATE BoardController DataHandler MLModule)

# copy
if (MSVC)
#    add_custom_command (TARGET BoardController POST_BUILD
#        # copy board controller libs
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/src/board_controller/inc/board_controller.h" "${CMAKE_HOME_DIRECTORY}/matlab-package/brainflow/inc/board_controller.h"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/src/board_controller/inc/board_info_getter.h" "${CMAKE_HOME_DIRECTORY}/matlab-package/brainflow/inc/board_info_getter.h"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/src/utils/inc/shared_export_matlab.h" "${CMAKE_HOME_DIRECTORY}/matlab-package/brainflow/inc/shared_export.h"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/third_party/neurosdk/lib/${NEUROSDK_LIB_NAME}.dll" "${CMAKE_HOME_DIRECTORY}/csharp-package/brainflow/brainflow/lib/${NEUROSDK_LIB_NAME}.dll"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/third_party/neurosdk/lib/${NEUROSDK_LIB_NAME}.dll" "${CMAKE_HOME_DIRECTORY}/python-package/brainflow/lib/${NEUROSDK_LIB_NAME}.dll"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/third_party/neurosdk/lib/${NEUROSDK_LIB_NAME}.dll" "${CMAKE_HOME_DIRECTORY}/julia-package/brainflow/lib/${NEUROSDK_LIB_NAME}.dll"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/third_party/neurosdk/lib/${NEUROSDK_LIB_NAME}.dll" "${CMAKE_HOME_DIRECTORY}/java-package/brainflow/src/main/resources/${NEUROSDK_LIB_NAME}.dll"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/third_party/neurosdk/lib/${NEUROSDK_LIB_NAME}.dll" "${CMAKE_HOME_DIRECTORY}/matlab-package/brainflow/lib/${NEUROSDK_LIB_NAME}.dll"
#    )
#    add_custom_command (TARGET DataHandler POST_BUILD
#        # copy data handler libs
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/compiled/$<CONFIG>/${DATA_HANDLER_COMPILED_NAME}" "${CMAKE_HOME_DIRECTORY}/python-package/brainflow/lib/${DATA_HANDLER_COMPILED_NAME}"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/compiled/$<CONFIG>/${DATA_HANDLER_COMPILED_NAME}" "${CMAKE_HOME_DIRECTORY}/julia-package/brainflow/lib/${DATA_HANDLER_COMPILED_NAME}"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/compiled/$<CONFIG>/${DATA_HANDLER_COMPILED_NAME}" "${CMAKE_HOME_DIRECTORY}/java-package/brainflow/src/main/resources/${DATA_HANDLER_COMPILED_NAME}"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/compiled/$<CONFIG>/${DATA_HANDLER_COMPILED_NAME}" "${CMAKE_HOME_DIRECTORY}/csharp-package/brainflow/brainflow/lib/${DATA_HANDLER_COMPILED_NAME}"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/compiled/$<CONFIG>/${DATA_HANDLER_COMPILED_NAME}" "${CMAKE_HOME_DIRECTORY}/matlab-package/brainflow/lib/${DATA_HANDLER_COMPILED_NAME}"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/src/data_handler/inc/data_handler.h" "${CMAKE_HOME_DIRECTORY}/matlab-package/brainflow/inc/data_handler.h"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/src/utils/inc/shared_export_matlab.h" "${CMAKE_HOME_DIRECTORY}/matlab-package/brainflow/inc/shared_export.h"
#    )
#    add_custom_command (TARGET MLModule POST_BUILD
#        # copy ml module libs
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/compiled/$<CONFIG>/${ML_MODULE_COMPILED_NAME}" "${CMAKE_HOME_DIRECTORY}/python-package/brainflow/lib/${ML_MODULE_COMPILED_NAME}"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/compiled/$<CONFIG>/${ML_MODULE_COMPILED_NAME}" "${CMAKE_HOME_DIRECTORY}/julia-package/brainflow/lib/${ML_MODULE_COMPILED_NAME}"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/src/ml/train/brainflow_svm.model" "${CMAKE_HOME_DIRECTORY}/python-package/brainflow/lib/brainflow_svm.model"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/compiled/$<CONFIG>/${ML_MODULE_COMPILED_NAME}" "${CMAKE_HOME_DIRECTORY}/java-package/brainflow/src/main/resources/${ML_MODULE_COMPILED_NAME}"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/src/ml/train/brainflow_svm.model" "${CMAKE_HOME_DIRECTORY}/java-package/brainflow/src/main/resources/brainflow_svm.model"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/compiled/$<CONFIG>/${ML_MODULE_COMPILED_NAME}" "${CMAKE_HOME_DIRECTORY}/csharp-package/brainflow/brainflow/lib/${ML_MODULE_COMPILED_NAME}"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/src/ml/train/brainflow_svm.model" "${CMAKE_HOME_DIRECTORY}/csharp-package/brainflow/brainflow/lib/brainflow_svm.model"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/compiled/$<CONFIG>/${ML_MODULE_COMPILED_NAME}" "${CMAKE_HOME_DIRECTORY}/matlab-package/brainflow/lib/${ML_MODULE_COMPILED_NAME}"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/src/ml/inc/ml_module.h" "${CMAKE_HOME_DIRECTORY}/matlab-package/brainflow/inc/ml_module.h"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/src/utils/inc/shared_export_matlab.h" "${CMAKE_HOME_DIRECTORY}/matlab-package/brainflow/inc/shared_export.h"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/src/ml/train/brainflow_svm.model" "${CMAKE_HOME_DIRECTORY}/matlab-package/brainflow/lib/brainflow_svm.model"
#    )
#    # copy Unicorn.dll
#    add_custom_command (TARGET BoardController POST_BUILD
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/third_party/unicorn/lib/Unicorn.dll" "${CMAKE_HOME_DIRECTORY}/csharp-package/brainflow/brainflow/lib/Unicorn.dll"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/third_party/unicorn/lib/Unicorn.dll" "${CMAKE_HOME_DIRECTORY}/python-package/brainflow/lib/Unicorn.dll"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/third_party/unicorn/lib/Unicorn.dll" "${CMAKE_HOME_DIRECTORY}/julia-package/brainflow/lib/Unicorn.dll"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/third_party/unicorn/lib/Unicorn.dll" "${CMAKE_HOME_DIRECTORY}/java-package/brainflow/src/main/resources/Unicorn.dll"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/third_party/unicorn/lib/Unicorn.dll" "${CMAKE_HOME_DIRECTORY}/matlab-package/brainflow/lib/Unicorn.dll"
#    )
endif (MSVC)
if (UNIX AND NOT ANDROID)
#    add_custom_command (TARGET BoardController POST_BUILD
#        # copy board controller libs
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/compiled/${BOARD_CONTROLLER_COMPILED_NAME}" "${CMAKE_HOME_DIRECTORY}/python-package/brainflow/lib/${BOARD_CONTROLLER_COMPILED_NAME}"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/compiled/${BOARD_CONTROLLER_COMPILED_NAME}" "${CMAKE_HOME_DIRECTORY}/julia-package/brainflow/lib/${BOARD_CONTROLLER_COMPILED_NAME}"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/compiled/${BOARD_CONTROLLER_COMPILED_NAME}" "${CMAKE_HOME_DIRECTORY}/java-package/brainflow/src/main/resources/${BOARD_CONTROLLER_COMPILED_NAME}"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/compiled/${BOARD_CONTROLLER_COMPILED_NAME}" "${CMAKE_HOME_DIRECTORY}/csharp-package/brainflow/brainflow/${BOARD_CONTROLLER_COMPILED_NAME}"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/src/board_controller/inc/board_controller.h" "${CMAKE_HOME_DIRECTORY}/matlab-package/brainflow/inc/board_controller.h"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/src/board_controller/inc/board_info_getter.h" "${CMAKE_HOME_DIRECTORY}/matlab-package/brainflow/inc/board_info_getter.h"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/src/utils/inc/shared_export_matlab.h" "${CMAKE_HOME_DIRECTORY}/matlab-package/brainflow/inc/shared_export.h"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/src/utils/inc/brainflow_constants.h" "${CMAKE_HOME_DIRECTORY}/matlab-package/brainflow/inc/brainflow_constants.h"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/compiled/${BOARD_CONTROLLER_COMPILED_NAME}" "${CMAKE_HOME_DIRECTORY}/matlab-package/brainflow/lib/${BOARD_CONTROLLER_COMPILED_NAME}"
#    )
#    add_custom_command (TARGET DataHandler POST_BUILD
#        # copy data handler libs
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/compiled/${DATA_HANDLER_COMPILED_NAME}" "${CMAKE_HOME_DIRECTORY}/python-package/brainflow/lib/${DATA_HANDLER_COMPILED_NAME}"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/compiled/${DATA_HANDLER_COMPILED_NAME}" "${CMAKE_HOME_DIRECTORY}/julia-package/brainflow/lib/${DATA_HANDLER_COMPILED_NAME}"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/compiled/${DATA_HANDLER_COMPILED_NAME}" "${CMAKE_HOME_DIRECTORY}/java-package/brainflow/src/main/resources/${DATA_HANDLER_COMPILED_NAME}"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/compiled/${DATA_HANDLER_COMPILED_NAME}" "${CMAKE_HOME_DIRECTORY}/csharp-package/brainflow/brainflow/lib/${DATA_HANDLER_COMPILED_NAME}"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/src/data_handler/inc/data_handler.h" "${CMAKE_HOME_DIRECTORY}/matlab-package/brainflow/inc/data_handler.h"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/src/utils/inc/shared_export_matlab.h" "${CMAKE_HOME_DIRECTORY}/matlab-package/brainflow/inc/shared_export.h"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/compiled/${DATA_HANDLER_COMPILED_NAME}" "${CMAKE_HOME_DIRECTORY}/matlab-package/brainflow/lib/${DATA_HANDLER_COMPILED_NAME}"
#    )
#    add_custom_command (TARGET MLModule POST_BUILD
#        # copy ml libs
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/compiled/${ML_MODULE_COMPILED_NAME}" "${CMAKE_HOME_DIRECTORY}/python-package/brainflow/lib/${ML_MODULE_COMPILED_NAME}"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/compiled/${ML_MODULE_COMPILED_NAME}" "${CMAKE_HOME_DIRECTORY}/julia-package/brainflow/lib/${ML_MODULE_COMPILED_NAME}"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/src/ml/train/brainflow_svm.model" "${CMAKE_HOME_DIRECTORY}/python-package/brainflow/lib/brainflow_svm.model"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/compiled/${ML_MODULE_COMPILED_NAME}" "${CMAKE_HOME_DIRECTORY}/java-package/brainflow/src/main/resources/${ML_MODULE_COMPILED_NAME}"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/src/ml/train/brainflow_svm.model" "${CMAKE_HOME_DIRECTORY}/java-package/brainflow/src/main/resources/brainflow_svm.model"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/compiled/${ML_MODULE_COMPILED_NAME}" "${CMAKE_HOME_DIRECTORY}/csharp-package/brainflow/brainflow/lib/${ML_MODULE_COMPILED_NAME}"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/src/ml/train/brainflow_svm.model" "${CMAKE_HOME_DIRECTORY}/csharp-package/brainflow/brainflow/lib/brainflow_svm.model"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/src/ml/inc/ml_module.h" "${CMAKE_HOME_DIRECTORY}/matlab-package/brainflow/inc/ml_module.h"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/src/utils/inc/shared_export_matlab.h" "${CMAKE_HOME_DIRECTORY}/matlab-package/brainflow/inc/shared_export.h"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/compiled/${ML_MODULE_COMPILED_NAME}" "${CMAKE_HOME_DIRECTORY}/matlab-package/brainflow/lib/${ML_MODULE_COMPILED_NAME}"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/src/ml/train/brainflow_svm.model" "${CMAKE_HOME_DIRECTORY}/matlab-package/brainflow/lib/brainflow_svm.model"
#    )
endif (UNIX AND NOT ANDROID)
if (UNIX AND NOT APPLE AND NOT ANDROID)
#    # copy libunicorn.so
#    add_custom_command (TARGET BoardController POST_BUILD 
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/third_party/unicorn/lib/libunicorn.so" "${CMAKE_HOME_DIRECTORY}/csharp-package/brainflow/brainflow/lib/libunicorn.so"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/third_party/unicorn/lib/libunicorn.so" "${CMAKE_HOME_DIRECTORY}/python-package/brainflow/lib/libunicorn.so"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/third_party/unicorn/lib/libunicorn.so" "${CMAKE_HOME_DIRECTORY}/julia-package/brainflow/lib/libunicorn.so"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/third_party/unicorn/lib/libunicorn.so" "${CMAKE_HOME_DIRECTORY}/java-package/brainflow/src/main/resources/libunicorn.so"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/third_party/unicorn/lib/libunicorn.so" "${CMAKE_HOME_DIRECTORY}/matlab-package/brainflow/lib/linunicorn.so"
#    )
endif (UNIX AND NOT APPLE AND NOT ANDROID)
if (APPLE)
#    add_custom_command (TARGET BoardController POST_BUILD
#        # copy neurosdk lib for MacOS
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/third_party/neurosdk/lib/lib${NEUROSDK_LIB_NAME}.dylib" "${CMAKE_HOME_DIRECTORY}/csharp-package/brainflow/brainflow/lib/lib${NEUROSDK_LIB_NAME}.dylib"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/third_party/neurosdk/lib/lib${NEUROSDK_LIB_NAME}.dylib" "${CMAKE_HOME_DIRECTORY}/python-package/brainflow/lib/lib${NEUROSDK_LIB_NAME}.dylib"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/third_party/neurosdk/lib/lib${NEUROSDK_LIB_NAME}.dylib" "${CMAKE_HOME_DIRECTORY}/julia-package/brainflow/lib/lib${NEUROSDK_LIB_NAME}.dylib"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/third_party/neurosdk/lib/lib${NEUROSDK_LIB_NAME}.dylib" "${CMAKE_HOME_DIRECTORY}/java-package/brainflow/src/main/resources/lib${NEUROSDK_LIB_NAME}.dylib"
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/third_party/neurosdk/lib/lib${NEUROSDK_LIB_NAME}.dylib" "${CMAKE_HOME_DIRECTORY}/matlab-package/brainflow/lib/lib${NEUROSDK_LIB_NAME}.dylib"
#    )
endif (APPLE)

if (ANDROID)
#    add_custom_command (TARGET BoardController POST_BUILD
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/compiled/${BOARD_CONTROLLER_COMPILED_NAME}" "${CMAKE_HOME_DIRECTORY}/tools/jniLibs/${ANDROID_ABI}/${BOARD_CONTROLLER_COMPILED_NAME}"
#    )
    if (LibFTDI1_FOUND) # this 'if. could be removed if more dependency libraries exist
#        add_custom_command (TARGET BoardController POST_BUILD
#            COMMAND "${CMAKE_COMMAND}" -E make_directory "${CMAKE_HOME_DIRECTORY}/java-package/android/src/main/libs/${ANDROID_ABI}/"
#        )
        foreach (DEPENDENCY_LIBRARY ${LIBFTDI_LIBRARIES})
            string (REGEX REPLACE "^-l" "" DEPENDENCY_LIBRARY ${DEPENDENCY_LIBRARY})
            if (DEPENDENCY_LIBRARY STREQUAL "log")
                continue ()
            endif (DEPENDENCY_LIBRARY STREQUAL "log")
            find_library (DEPENDENCY_LIBRARY_LOCATION NAMES ${DEPENDENCY_LIBRARY} HINTS ${LIBFTDI_LIBRARY_DIRS} / NO_CMAKE_PATH NO_CMAKE_FIND_ROOT_PATH)
#            add_custom_command (TARGET BoardController POST_BUILD
#                COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${DEPENDENCY_LIBRARY_LOCATION}" "${CMAKE_HOME_DIRECTORY}/tools/jniLibs/${ANDROID_ABI}/"
#                COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${DEPENDENCY_LIBRARY_LOCATION}" "${CMAKE_HOME_DIRECTORY}/java-package/android/src/main/libs/${ANDROID_ABI}/"
#            )
            unset (DEPENDENCY_LIBRARY_LOCATION CACHE)
        endforeach (DEPENDENCY_LIBRARY)
    endif (LibFTDI1_FOUND)
#    add_custom_command (TARGET DataHandler POST_BUILD
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/compiled/${DATA_HANDLER_COMPILED_NAME}" "${CMAKE_HOME_DIRECTORY}/tools/jniLibs/${ANDROID_ABI}/${DATA_HANDLER_COMPILED_NAME}"
#    )
#    add_custom_command (TARGET MLModule POST_BUILD
#        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/compiled/${ML_MODULE_COMPILED_NAME}" "${CMAKE_HOME_DIRECTORY}/tools/jniLibs/${ANDROID_ABI}/${ML_MODULE_COMPILED_NAME}"
#    )
endif (ANDROID)

if (CMAKE_SIZEOF_VOID_P EQUAL 8)
    MESSAGE ("64 bits compiler detected")
    SET (PLATFORM_ACH "X64")
#    set_property (TARGET BoardController PROPERTY OUTPUT_NAME BoardController64)
#    set_property (TARGET DataHandler PROPERTY OUTPUT_NAME DataHandler64)
#    set_property (TARGET MLModule PROPERTY OUTPUT_NAME MLModule64)
    if(APPLE)
        SET (NEUROSDK_LIB_NAME "neurosdk-shared")
    elseif (UNIX)
    else ()
        SET (NEUROSDK_LIB_NAME "neurosdk-x64")
    endif (APPLE)
else (CMAKE_SIZEOF_VOID_P EQUAL 8)
    MESSAGE ("32 bits compiler detected")
    SET (PLATFORM_ACH "X86")
    # for linux and apple we dont add 32bit library to packages, so there can not be a conflict between libs
    # to dont write code like if(32 bits) if(64 bits) in bindings and keep the ability to use them if brainflow was built from src
    # use the same name for 32 and 64 bits
    set_property (TARGET BoardController PROPERTY OUTPUT_NAME BoardController32)
    set_property (TARGET DataHandler PROPERTY OUTPUT_NAME DataHandler32)
    set_property (TARGET MLModule PROPERTY OUTPUT_NAME MLModule32)
    if (APPLE)
    elseif (UNIX)
    else ()
        SET (NEUROSDK_LIB_NAME "neurosdk-x86")
    endif (APPLE)
endif (CMAKE_SIZEOF_VOID_P EQUAL 8)

# install
set (VERSION_CONFIG ${CMAKE_HOME_DIRECTORY}/compiled/brainflowConfigVersion.cmake)
set (CONFIG_INSTALL_DIR lib/cmake/brainflow)
set (PROJECT_CONFIG ${CMAKE_HOME_DIRECTORY}/compiled/brainflowConfig.cmake)
set (TARGETS_EXPORT_NAME brainflowTargets)
set (PACKAGE_LIB_INSTALL_DIR ${CMAKE_INSTALL_PREFIX}/lib)
set (PACKAGE_INC_INSTALL_DIR ${CMAKE_INSTALL_PREFIX}/include)

include (CMakePackageConfigHelpers)

write_basic_package_version_file (
    ${VERSION_CONFIG}
    VERSION ${BRAINFLOW_VERSION}
    COMPATIBILITY AnyNewerVersion
)

configure_package_config_file (
    cmake/Config.cmake.in
    ${PROJECT_CONFIG}
    INSTALL_DESTINATION ${CONFIG_INSTALL_DIR}
)

# install svm_model
install (
    FILES
    ${CMAKE_HOME_DIRECTORY}/src/ml/train/brainflow_svm.model
    DESTINATION lib
)

# install 3rd party deps
if (MSVC)
#    install (
#        FILES
#        ${CMAKE_HOME_DIRECTORY}/third_party/neurosdk/lib/${NEUROSDK_LIB_NAME}.dll
#        DESTINATION lib
#    )
    if (CMAKE_SIZEOF_VOID_P EQUAL 8 AND BUILD_OYMOTION_SDK)
#        install (
#            FILES
#            ${CMAKE_HOME_DIRECTORY}/third_party/gForceSDKCXX/compiled/$<CONFIG>/gForceSDKWrapper.dll
#            ${CMAKE_HOME_DIRECTORY}/third_party/gForceSDKCXX/lib/gforce64.dll
#            DESTINATION lib
#        )
    endif (CMAKE_SIZEOF_VOID_P EQUAL 8 AND BUILD_OYMOTION_SDK)
    if (CMAKE_SIZEOF_VOID_P EQUAL 4 AND BUILD_OYMOTION_SDK)
#        install (
#            FILES
#            ${CMAKE_HOME_DIRECTORY}/third_party/gForceSDKCXX/compiled/$<CONFIG>/gForceSDKWrapper32.dll
#            ${CMAKE_HOME_DIRECTORY}/third_party/gForceSDKCXX/lib/gforce32.dll
#            DESTINATION lib
#        )
    endif (CMAKE_SIZEOF_VOID_P EQUAL 4 AND BUILD_OYMOTION_SDK)
endif (MSVC)

if (APPLE)
#    install (
#        FILES
#        ${CMAKE_HOME_DIRECTORY}/third_party/neurosdk/lib/lib${NEUROSDK_LIB_NAME}.dylib
#        DESTINATION lib
#    )
endif (APPLE)

# install unicorn lib
if (UNIX AND NOT APPLE)
#    install (
#        FILES
#        ${CMAKE_HOME_DIRECTORY}/third_party/unicorn/lib/libunicorn.so
#        DESTINATION lib
#    )
endif (UNIX AND NOT APPLE)

if (MSVC)
#    install (
#        FILES
#        ${CMAKE_HOME_DIRECTORY}/third_party/unicorn/lib/Unicorn.dll
#        DESTINATION lib
#    )
endif (MSVC)

install (
    TARGETS BoardController DataHandler ${DSPFILTERS} ${WAVELIB} MLModule Brainflow
    EXPORT ${TARGETS_EXPORT_NAME}
    RUNTIME DESTINATION lib
    LIBRARY DESTINATION lib
    INCLUDES DESTINATION include
    PUBLIC_HEADER DESTINATION include
    ARCHIVE DESTINATION lib
)

install (
    FILES ${PROJECT_CONFIG} ${VERSION_CONFIG}
    DESTINATION ${CONFIG_INSTALL_DIR}
)

install (
    EXPORT ${TARGETS_EXPORT_NAME}
    NAMESPACE brainflow::
    DESTINATION ${CONFIG_INSTALL_DIR}
)

install (
    TARGETS BoardController DataHandler MLModule
    EXPORT python-package
    RUNTIME DESTINATION ${CMAKE_HOME_DIRECTORY}/python-package/brainflow/lib
    LIBRARY DESTINATION ${CMAKE_HOME_DIRECTORY}/python-package/brainflow/lib
    ARCHIVE DESTINATION ${CMAKE_HOME_DIRECTORY}/python-package/brainflow/lib
    EXCLUDE_FROM_ALL
)

install (
    TARGETS BoardController DataHandler MLModule
    EXPORT java-package
    RUNTIME DESTINATION ${CMAKE_HOME_DIRECTORY}/java-package/brainflow/src/main/resources
    LIBRARY DESTINATION ${CMAKE_HOME_DIRECTORY}/java-package/brainflow/src/main/resources
    ARCHIVE DESTINATION ${CMAKE_HOME_DIRECTORY}/java-package/brainflow/src/main/resources
    EXCLUDE_FROM_ALL
)

install (
    TARGETS BoardController DataHandler MLModule
    EXPORT csharp-package
    RUNTIME DESTINATION ${CMAKE_HOME_DIRECTORY}/csharp-package/brainflow/brainflow/lib
    LIBRARY DESTINATION ${CMAKE_HOME_DIRECTORY}/csharp-package/brainflow/brainflow/lib
    ARCHIVE DESTINATION ${CMAKE_HOME_DIRECTORY}/csharp-package/brainflow/brainflow/lib
    EXCLUDE_FROM_ALL
)

install (
    TARGETS BoardController DataHandler MLModule
    EXPORT matlab-package
    RUNTIME DESTINATION ${CMAKE_HOME_DIRECTORY}/matlab-package/brainflow/lib
    LIBRARY DESTINATION ${CMAKE_HOME_DIRECTORY}/matlab-package/brainflow/lib
    INCLUDES DESTINATION ${CMAKE_HOME_DIRECTORY}/matlab-package/brainflow/inc
    PUBLIC_HEADER DESTINATION ${CMAKE_HOME_DIRECTORY}/matlab-package/brainflow/inc
    ARCHIVE DESTINATION ${CMAKE_HOME_DIRECTORY}/matlab-package/brainflow/lib
    EXCLUDE_FROM_ALL
)

install (
    FILES src/utils/inc/shared_export_matlab.h
    RENAME shared_export.h
    DESTINATION ${CMAKE_HOME_DIRECTORY}/matlab-package/brainflow/inc
    EXCLUDE_FROM_ALL
)

install (
    TARGETS BoardController DataHandler MLModule
    EXPORT julia-package
    RUNTIME DESTINATION ${CMAKE_HOME_DIRECTORY}/julia-package_/brainflow/lib
    LIBRARY DESTINATION ${CMAKE_HOME_DIRECTORY}/julia-package_/brainflow/lib
    ARCHIVE DESTINATION ${CMAKE_HOME_DIRECTORY}/julia-package_/brainflow/lib
)