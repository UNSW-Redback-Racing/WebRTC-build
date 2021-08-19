cmake_minimum_required(VERSION 3.11)

project(WebRTC VERSION 0.1.1)

include(CMakePackageConfigHelpers)

set(CMAKE_MODULE_PATH
	${CMAKE_MODULE_PATH}
	${CMAKE_SOURCE_DIR}/cmake)

find_package(DepotTools REQUIRED)
find_package(Git REQUIRED)

# Configuration steps

set(fetch_cmd fetch --no-history --force --nohooks webrtc)

if(WIN32)
	set(fetch_cmd cmd /c ${fetch_cmd})
endif()


add_custom_command(
	OUTPUT ${CMAKE_BINARY_DIR}/.fetch_rslt
	COMMAND ${fetch_cmd}
)

add_custom_target(
	webrtc_fetch ALL
	DEPENDS ${CMAKE_BINARY_DIR}/.fetch_rslt
)

set(webrtc_release_branch cd src && git checkout -B rel-4280 refs/remotes/branch-heads/4280)

if(WIN32)
	set(webrtc_release_branch cmd /c ${webrtc_release_branch})
endif()

add_custom_command(
	OUTPUT ${CMAKE_BINARY_DIR}/rbranch
	COMMAND ${webrtc_release_branch}
)

add_custom_target(
	webrtc_release_branch ALL
	DEPENDS ${CMAKE_BINARY_DIR}/rbranch
)

add_dependencies(webrtc_release_branch webrtc_fetch)
#  ./build/install-build-deps.sh --quick-check --no-prompt --no-chromeos-fonts
set(webrtc_release_sync cd src && gclient sync -D)

if(WIN32)
	set(webrtc_release_sync cmd /c ${webrtc_release_sync})
endif()

add_custom_command(
	OUTPUT ${CMAKE_BINARY_DIR}/relsync
	COMMAND ${webrtc_release_sync}
)

add_custom_target(
	webrtc_release_sync ALL
	DEPENDS ${CMAKE_BINARY_DIR}/relsync
)

add_dependencies(webrtc_release_sync webrtc_release_branch)


# Making sure to install the arm sysroot
if (DEFINED TARGET_CPU)
	set(webrtc_arm_sysroot cd ./src/build/linux/sysroot_scripts && ./install-sysroot.py --arch=${TARGET_CPU})
    
	if(WIN32)
		set(webrtc_arm_sysroot cmd /c ${webrtc_arm_sysroot})
	endif()

	add_custom_command(
		OUTPUT ${CMAKE_BINARY_DIR}/arm_sysroot
		COMMAND ${webrtc_arm_sysroot}
	)
	
	add_custom_target(
		webrtc_arm_sysroot ALL
		DEPENDS ${CMAKE_BINARY_DIR}/arm_sysroot
	)

	add_dependencies(webrtc_arm_sysroot webrtc_release_branch)
	
	set(prebuild_step webrtc_arm_sysroot)
else()
	set(prebuild_step webrtc_release_branch)
endif()

if (NOT CMAKE_BUILD_TYPE)
	set(CMAKE_BUILD_TYPE Debug)
endif()


# debug or release build
set(BUILD_TYPE ${CMAKE_BUILD_TYPE})
if (BUILD_TYPE STREQUAL Release)
	message(STATUS "Generating Release build")
	set(BUILD_TYPE "is_debug=false")
	set(LibName libwebrtc)
else()
	set(BUILD_TYPE "")
	set(LibName libwebrtcd)
endif()


if (DEFINED TARGET_OS)
	message(STATUS "Setting target os to ${TARGET_OS}")
	set(TARGET_OS_ARG "target_os=\"${TARGET_OS}\"")
endif()

if (DEFINED TARGET_CPU)
	message(STATUS "Setting target cpu to ${TARGET_CPU}")
	set(TARGET_CPU_ARG "target_cpu=\"${TARGET_CPU}\"")
endif()

if (DEFINED CUSTOM_TOOLCHAIN)
	message(STATUS "Setting custom toolchain to ${CUSTOM_TOOLCHAIN}")
	set(CUSTOM_TOOLCHAIN_ARG "custom_toolchain=\"${CUSTOM_TOOLCHAIN}\"")
endif()

if (TARGET_CPU STREQUAL arm AND ARM_VERSION STREQUAL 6)
		message(STATUS "Setting ${TARGET_CPU} version to ${ARM_VERSION}")
		set(ARM_VERSION_ARG "arm_version=${ARM_VERSION}")
		list(APPEND ARM_VERSION_ARG 
			arm_float_abi="hard"
			arm_use_neon=false
			arm_arch="armv6"
			arm_fpu="vfp"
			arm_tune="arm1176jzf-s"
			enable_libaom=false
			rtc_build_tools=true
			treat_warnings_as_errors=false
			fatal_linker_warnings=false
		)
endif()
set(webrtc_gen cd src && gn gen ${CMAKE_BINARY_DIR}/src/out/${CMAKE_BUILD_TYPE} 
		--args='${BUILD_TYPE} ${TARGET_OS_ARG} ${TARGET_CPU_ARG} ${ARM_VERSION_ARG} ${CUSTOM_TOOLCHAIN_ARG} use_rtti=true rtc_include_tests=false rtc_enable_protobuf=false use_custom_libcxx=false use_custom_libcxx_for_host=false use_x11=false is_component_build=false enable_dsyms=true')

add_custom_command(
		OUTPUT ${CMAKE_BINARY_DIR}/src/out/${CMAKE_BUILD_TYPE}/all
		COMMAND ${webrtc_gen}
)

add_custom_target(
	webrtc_gen ALL
	DEPENDS ${CMAKE_BINARY_DIR}/src/out/${CMAKE_BUILD_TYPE}/all
)

add_dependencies(webrtc_gen ${prebuild_step})

set(webrtc_build ninja -C ${CMAKE_BINARY_DIR}/src/out/${CMAKE_BUILD_TYPE})

add_custom_command(
		OUTPUT ${CMAKE_BINARY_DIR}/src/out/${CMAKE_BUILD_TYPE}/${LibName}.${WEBRTC_LIB_EXT}
		COMMAND ${webrtc_build}
)


set(WEBRTC_LIBS ${CMAKE_BINARY_DIR}/src/out/${CMAKE_BUILD_TYPE}/obj/${LibName}.${WEBRTC_LIB_EXT})
set(WEBRTC_INCLUDE_DIR ${CMAKE_BINARY_DIR}/src)

add_custom_target(
	webrtc_build ALL
	DEPENDS ${CMAKE_BINARY_DIR}/src/out/${CMAKE_BUILD_TYPE}/${LibName}.${WEBRTC_LIB_EXT}
)

add_dependencies(webrtc_build webrtc_gen)


include(Install)
include(Package)
