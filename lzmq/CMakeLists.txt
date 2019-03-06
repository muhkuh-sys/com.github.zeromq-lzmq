cmake_minimum_required(VERSION 3.2.3)

PROJECT("lzmq")

INCLUDE(ExternalProject)

SET(VERSION_EXTERNAL "0.4.4")
SET(VERSION_PACKAGE  "1")
SET(PROJECT_VERSION  "${VERSION_EXTERNAL}.${VERSION_PACKAGE}")

# Get the VCS version for the jonchki configuration.
INCLUDE(${CMAKE_HOME_DIRECTORY}/cmake/version.cmake)

# Filter the jonchki configuration.
CONFIGURE_FILE(installer/lua${BUILDCFG_LUA_VERSION}/lzmq.xml
               ${CMAKE_BINARY_DIR}/lua${BUILDCFG_LUA_VERSION}-lzmq-${PROJECT_VERSION}.xml
               @ONLY)


#----------------------------------------------------------------------------
#
# Build the project.
#

IF((${BUILDCFG_ONLY_JONCHKI_CFG} STREQUAL "OFF"))
	# Select the LUA version.
	INCLUDE(${CMAKE_HOME_DIRECTORY}/cmake/muhkuh_lua.cmake)

	# Python is used for the hashes and the test scripts.
	FIND_PACKAGE(PythonInterp 2.7 REQUIRED)

	SET(SUB_CMAKE_ARGS "")
	LIST(APPEND SUB_CMAKE_ARGS "-DCMAKE_C_FLAGS=${CMAKE_C_FLAGS}")
	LIST(APPEND SUB_CMAKE_ARGS "-DCMAKE_SYSTEM_NAME=${CMAKE_SYSTEM_NAME}")
	LIST(APPEND SUB_CMAKE_ARGS "-DCMAKE_C_COMPILER=${CMAKE_C_COMPILER}")
	LIST(APPEND SUB_CMAKE_ARGS "-DCMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER}")
	IF(${CMAKE_SYSTEM_NAME} STREQUAL "Windows")
		LIST(APPEND SUB_CMAKE_ARGS "-DCMAKE_RC_COMPILER=${CMAKE_RC_COMPILER}")
	ENDIF(${CMAKE_SYSTEM_NAME} STREQUAL "Windows")
	LIST(APPEND SUB_CMAKE_ARGS "-DLUA_LIBRARIES=${LUA_LIBRARIES}")
	LIST(APPEND SUB_CMAKE_ARGS "-DLUA_LIBRARY=${LUA_LIBRARIES}")
	LIST(APPEND SUB_CMAKE_ARGS "-DLUA_INCLUDE_DIR=${LUA_INCLUDE_DIR}")
	LIST(APPEND SUB_CMAKE_ARGS "-DCMAKE_INSTALL_PREFIX=''")
	LIST(APPEND SUB_CMAKE_ARGS "-DZMQ_LIBRARY=${CMAKE_BINARY_DIR}/build_requirements/jonchki/install/dev/lib/libzmq.a")
	LIST(APPEND SUB_CMAKE_ARGS "-DZMQ_INCLUDE_DIR=${CMAKE_BINARY_DIR}/build_requirements/jonchki/install/dev/include/")

	MESSAGE("ZMQ_LIBRARY=${CMAKE_CURRENT_BINARY_DIR}/build_requirements/jonchki/install/dev/lib/libzmq.a")
	MESSAGE("ZMQ_INCLUDE_DIR=${CMAKE_CURRENT_BINARY_DIR}/build_requirements/jonchki/install/dev/include/")

	ExternalProject_Add(TARGET_lzmq
	                    PREFIX ${CMAKE_CURRENT_BINARY_DIR}/lzmq-${VERSION_EXTERNAL}
	                    URL ${CMAKE_CURRENT_SOURCE_DIR}/lzmq-v${VERSION_EXTERNAL}.tar.gz
	                    URL_HASH SHA1=de3a5d1707caf234ca39e50694abb009d90fe318
	                    PATCH_COMMAND "${PYTHON_EXECUTABLE}" ${CMAKE_HOME_DIRECTORY}/cmake/tools/apply_patches.py --working-folder ${CMAKE_CURRENT_BINARY_DIR}/lzmq-${VERSION_EXTERNAL}/src/TARGET_lzmq --patch-folder ${CMAKE_CURRENT_SOURCE_DIR}/patches --strip 1
	                    CMAKE_ARGS ${SUB_CMAKE_ARGS}
	                    INSTALL_COMMAND ${CMAKE_MAKE_PROGRAM} install DESTDIR=${CMAKE_CURRENT_BINARY_DIR}/install
	)


	#----------------------------------------------------------------------------
	#
	# Build a package for the selected platform.
	#

	SET(CPACK_OUTPUT_FILE_PREFIX ${CMAKE_BINARY_DIR}/../../repository/com/github/zeromq/lzmq/${PROJECT_VERSION})
	SET(CPACK_INCLUDE_TOPLEVEL_DIRECTORY FALSE)
	SET(CPACK_PACKAGE_FILE_NAME "lua${BUILDCFG_LUA_VERSION}-lzmq-${PROJECT_VERSION}-${JONCHKI_PLATFORM}")

	SET(CPACK_GENERATOR "TXZ")
	INCLUDE(CPack)

	# This is the folder where the external project TARGET_bitop installed the files.
	SET(SUB_INSTALL_DIR "${CMAKE_CURRENT_BINARY_DIR}/install/lib/lua")

	IF(${JONCHKI_PLATFORM_DIST_ID} STREQUAL "windows")
		# Package a windows build.
		CONFIGURE_FILE(installer/lua${BUILDCFG_LUA_VERSION}/install_windows.lua
		               ${CMAKE_CURRENT_BINARY_DIR}/package/install.lua
		               @ONLY)

	ELSE(${JONCHKI_PLATFORM_DIST_ID} STREQUAL "windows")
		# Package a linux build.
		CONFIGURE_FILE(installer/lua${BUILDCFG_LUA_VERSION}/install_linux.lua
		               ${CMAKE_CURRENT_BINARY_DIR}/package/install.lua
		               @ONLY)
	ENDIF(${JONCHKI_PLATFORM_DIST_ID} STREQUAL "windows")

	INSTALL(FILES ${SUB_INSTALL_DIR}/lzmq${CMAKE_SHARED_MODULE_SUFFIX}
	        DESTINATION lua_plugins/)
	INSTALL(FILES ${SUB_INSTALL_DIR}/lzmq/timer${CMAKE_SHARED_MODULE_SUFFIX}
	        DESTINATION lua_plugins/lzmq/)

	INSTALL(FILES ${SUB_INSTALL_DIR}/lzmq/ffi.lua ${SUB_INSTALL_DIR}/lzmq/loop.lua ${SUB_INSTALL_DIR}/lzmq/poller.lua ${SUB_INSTALL_DIR}/lzmq/threads.lua
	        DESTINATION lua/lzmq/)
	INSTALL(FILES ${SUB_INSTALL_DIR}/lzmq/ffi/api.lua ${SUB_INSTALL_DIR}/lzmq/ffi/error.lua ${SUB_INSTALL_DIR}/lzmq/ffi/loop.lua ${SUB_INSTALL_DIR}/lzmq/ffi/poller.lua ${SUB_INSTALL_DIR}/lzmq/ffi/threads.lua ${SUB_INSTALL_DIR}/lzmq/ffi/timer.lua
	        DESTINATION lua/lzmq/ffi/)
	INSTALL(FILES ${SUB_INSTALL_DIR}/lzmq/impl/loop.lua ${SUB_INSTALL_DIR}/lzmq/impl/threads.lua
	        DESTINATION lua/lzmq/impl/)
	INSTALL(FILES ${SUB_INSTALL_DIR}/lzmq/llthreads/ex.lua
	        DESTINATION lua/lzmq/llthreads/)

	INSTALL(FILES ${CMAKE_CURRENT_BINARY_DIR}/package/install.lua
	        DESTINATION .)


	CONFIGURE_FILE(installer/lua${BUILDCFG_LUA_VERSION}/lzmq.xml
	               ${CPACK_OUTPUT_FILE_PREFIX}/lua${BUILDCFG_LUA_VERSION}-lzmq-${PROJECT_VERSION}-${JONCHKI_PLATFORM}.xml
	               @ONLY)


	#-----------------------------------------------------------------------------
	#
	# Create a custom target named "pack".
	# It builds the package and creates the hashes.
	#
	ADD_CUSTOM_TARGET(pack
	                  COMMAND ${CMAKE_MAKE_PROGRAM}
	                  COMMAND ${CMAKE_MAKE_PROGRAM} package
	                  COMMAND "${PYTHON_EXECUTABLE}" ${CMAKE_HOME_DIRECTORY}/cmake/tools/generate_hash.py ${CPACK_OUTPUT_FILE_PREFIX}/lua${BUILDCFG_LUA_VERSION}-lzmq-${PROJECT_VERSION}-${JONCHKI_PLATFORM}.xml
	                  COMMAND "${PYTHON_EXECUTABLE}" ${CMAKE_HOME_DIRECTORY}/cmake/tools/generate_hash.py ${CPACK_OUTPUT_FILE_PREFIX}/lua${BUILDCFG_LUA_VERSION}-lzmq-${PROJECT_VERSION}-${JONCHKI_PLATFORM}.tar.xz
	                  COMMENT "Pack the artifact and generate the hashes...")


	#-----------------------------------------------------------------------------
	#
	# Filter the pom.xml file.
	#
	SET(POM_ARTIFACT_ID  "lzmq")
	SET(POM_GROUP_ID     "com.github.zeromq")
	SET(POM_VERSION      "${PROJECT_VERSION}")
	SET(POM_PACKAGING    "tar.xz")
	CONFIGURE_FILE(installer/lua${BUILDCFG_LUA_VERSION}/pom.xml ${CPACK_OUTPUT_FILE_PREFIX}/lua${BUILDCFG_LUA_VERSION}-lzmq-${PROJECT_VERSION}.pom)
ENDIF((${BUILDCFG_ONLY_JONCHKI_CFG} STREQUAL "OFF"))
