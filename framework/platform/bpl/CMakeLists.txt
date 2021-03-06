project(bpl VERSION ${prplmesh_VERSION})

message("${BoldWhite}Preparing ${BoldGreen}${PROJECT_NAME}${BoldWhite} for the ${BoldGreen}${TARGET_PLATFORM}${BoldWhite} platform${ColourReset}")

# Set the base path for the current module
set(MODULE_PATH ${CMAKE_CURRENT_LIST_DIR})

# Common Sources
file(GLOB_RECURSE bpl_common_sources ${MODULE_PATH}/common/*.c*)

# Common prplmesh_db values
set(BEEROCKS_MANAGEMENT_MODE  "Multi-AP-Controller-and-Agent" CACHE STRING "Set management mode")
set(BEEROCKS_OPERATING_MODE  "Gateway" CACHE STRING "Set operating mode")
set(BEEROCKS_STOP_ON_FAILURE_ATTEMPTS  0 CACHE INT "In common config stop on failure disabled")
set(BEEROCKS_CERTIFICATION_MODE  1 CACHE INT "In certification, agent must retry onboarding indefinitely")

set(BEEROCKS_HOSTAP_WLAN1_CTRL_IFACE "/var/run/hostapd/${BEEROCKS_WLAN1_IFACE}" CACHE PATH "hostapd ctrl iface path for 1st WLAN")
set(BEEROCKS_HOSTAP_WLAN2_CTRL_IFACE "/var/run/hostapd/${BEEROCKS_WLAN2_IFACE}" CACHE PATH "hostapd ctrl iface path for 2nd WLAN")
set(BEEROCKS_HOSTAP_WLAN3_CTRL_IFACE "/var/run/hostapd/${BEEROCKS_WLAN3_IFACE}" CACHE PATH "hostapd ctrl iface path for 3rd WLAN")

set(BEEROCKS_WPA_SUPPLICANT_WLAN1_CTRL_IFACE "/var/run/wpa_supplicant/${BEEROCKS_WLAN1_IFACE}" CACHE PATH "wpa_supplicant ctrl iface path for 1st WLAN")
set(BEEROCKS_WPA_SUPPLICANT_WLAN2_CTRL_IFACE "/var/run/wpa_supplicant/${BEEROCKS_WLAN2_IFACE}" CACHE PATH "wpa_supplicant ctrl iface path for 2nd WLAN")
set(BEEROCKS_WPA_SUPPLICANT_WLAN3_CTRL_IFACE "/var/run/wpa_supplicant/${BEEROCKS_WLAN3_IFACE}" CACHE PATH "wpa_supplicant ctrl iface path for 3rd WLAN")

# Add prplmesh_platform_db config generation
add_subdirectory(platform_db)

# OpenWRT
if (TARGET_PLATFORM STREQUAL "openwrt")

    # UGW
    if (TARGET_PLATFORM_TYPE STREQUAL "ugw")

        # Platform definitions
        add_definitions(-DBEEROCKS_UGW)

        # Platform specific includes
        include_directories(${PLATFORM_INCLUDE_DIR}/libnl3)
        include_directories(${PLATFORM_INCLUDE_DIR})

        # libbridge
        file(GLOB LIBBRIDGE_SEARCH_PATHS "${PLATFORM_BUILD_DIR}/linux-intel_mips*/bridge-utils*/libbridge")
        find_path(LIBBRIDGE_INCLUDE_DIR NAMES "libbridge.h" PATHS "${LIBBRIDGE_SEARCH_PATHS}" NO_CMAKE_FIND_ROOT_PATH)
        include_directories(${LIBBRIDGE_INCLUDE_DIR})
        link_directories(${LIBBRIDGE_INCLUDE_DIR})

        # UCI BPL
        file(GLOB_RECURSE bpl_platform_sources ${MODULE_PATH}/uci/*.c*)

        # UGW specific libraries
        find_package(ubus REQUIRED)
        find_package(safec REQUIRED)

        link_directories(${PLATFORM_STAGING_DIR}/usr/lib)
        list(APPEND BPL_LIBS ubus nl-3 nl-route-3 bridge dl uci ubox safec)

        # Signal libsafec that we support the C99 standard
        add_definitions(-DHAVE_C99)
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -std=c99")

    # Other OpenWRT based platforms (e.g. Turris-Omnia)
    else()

        file(GLOB_RECURSE bpl_platform_sources ${MODULE_PATH}/linux/*.c*)

        add_definitions(-DPLATFORM_DB_PATH="${INSTALL_PATH}/share/prplmesh_platform_db")
        add_definitions(-DPLATFORM_DB_PATH_TEMP="${TMP_PATH}/prplmesh_platform_db")

    endif()

elseif (TARGET_PLATFORM STREQUAL "rdkb")

    # Platform definitions
    add_definitions(-DBEEROCKS_RDKB)

    # Platform specific includes
    include_directories(${PLATFORM_INCLUDE_DIR})
    include_directories(${PLATFORM_INCLUDE_DIR}/libnl3)
    include_directories(${PLATFORM_INCLUDE_DIR}/bridge-utils)

    # UCI BPL
    file(GLOB_RECURSE bpl_platform_sources ${MODULE_PATH}/uci/*.c*)

    # RDKB specific libraries
    find_package(ubus REQUIRED)
    find_package(slibc REQUIRED)

    list(APPEND BPL_LIBS nl-3 nl-route-3 bridge dl swpal uci ubox ubus slibc)

    # Install the UCI DB files
    file(GLOB_RECURSE bpl_uci_db_files ${MODULE_PATH}/uci/db/prplmesh_db*)
    install(FILES ${bpl_uci_db_files} DESTINATION ${CMAKE_INSTALL_PREFIX}/share)

elseif (TARGET_PLATFORM STREQUAL "linux")

    # Platform definitions
    add_definitions(-DBEEROCKS_LINUX)

    file(GLOB_RECURSE bpl_platform_sources ${MODULE_PATH}/linux/*.c*)

    add_definitions(-DPLATFORM_DB_PATH="${INSTALL_PATH}/share/prplmesh_platform_db")
    add_definitions(-DPLATFORM_DB_PATH_TEMP="${TMP_PATH}/prplmesh_platform_db")

else()

    message(WARNING "${BoldWhite}Skipping ${BoldGreen}${PROJECT_NAME}${BoldWhite} build for unsupported platform...${ColourReset}")
    return()

endif()

# Build the library
add_library(${PROJECT_NAME} ${bpl_common_sources} ${bpl_platform_sources})
set_target_properties(${PROJECT_NAME} PROPERTIES VERSION ${prplmesh_VERSION} SOVERSION ${prplmesh_VERSION_MAJOR})
set_target_properties(${PROJECT_NAME} PROPERTIES LINK_FLAGS "-Wl,-z,defs")
target_link_libraries(${PROJECT_NAME} PRIVATE elpp mapfcommon ${BPL_LIBS})

# Install
target_include_directories(${PROJECT_NAME}
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/include>
    )

install(TARGETS ${PROJECT_NAME}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    )

if(BUILD_TESTS)
  add_subdirectory(test)
endif()
