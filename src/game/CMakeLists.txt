#
# This file is part of the CMaNGOS Project. See AUTHORS file for Copyright information
#
# This program is free software; you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation; either version 2 of the License, or
# (at your option) any later version.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with this program; if not, write to the Free Software
# Foundation, Inc., 59 Temple Place, Suite 330, Boston, MA  02111-1307  USA
#

set(LIBRARY_NAME game)

add_definitions(-DDT_POLYREF64)
add_definitions(-D__ASSERT_MACROS_DEFINE_VERSIONS_WITHOUT_UNDERSCORES=0)

# Find all the input files
FILE(GLOB_RECURSE LIBRARY_SRCS RELATIVE "${CMAKE_CURRENT_SOURCE_DIR}" "*.h" "*.cpp")

if(NOT BUILD_SCRIPTDEV)
  # exclude script folder for faster build time (used for dev mode)
  set (EXCLUDE_DIR "AI/ScriptDevAI/scripts/")
  foreach (TMP_PATH ${LIBRARY_SRCS})
      string (FIND ${TMP_PATH} ${EXCLUDE_DIR} EXCLUDE_DIR_FOUND)
      if (NOT ${EXCLUDE_DIR_FOUND} EQUAL -1)
          list(REMOVE_ITEM LIBRARY_SRCS ${TMP_PATH})
      endif ()
  endforeach()
endif()

if(NOT BUILD_PLAYERBOT)
  # exclude Playerbot folder
  set (EXCLUDE_DIR "PlayerBot/")
  foreach (TMP_PATH ${LIBRARY_SRCS})
      string (FIND ${TMP_PATH} ${EXCLUDE_DIR} EXCLUDE_DIR_FOUND)
      if (NOT ${EXCLUDE_DIR_FOUND} EQUAL -1)
          list(REMOVE_ITEM LIBRARY_SRCS ${TMP_PATH})
      endif ()
  endforeach()
endif()

# Create visual studio groups using folder structure
foreach(source ${LIBRARY_SRCS})
    get_filename_component(source_path "${source}" PATH)
    string(REPLACE "/" "\\" source_path_msvc "${source_path}")
    source_group("${source_path_msvc}" FILES "${source}")
endforeach()

add_library(${LIBRARY_NAME} STATIC
  ${LIBRARY_SRCS}
)

target_link_libraries(${LIBRARY_NAME}
  PRIVATE shared
  PRIVATE g3dlite
  PRIVATE detour
  PRIVATE ${ZLIB_LIBRARIES}
  PRIVATE lualib
)

# include additionals headers
set(ADDITIONAL_INCLUDE_DIRS
  ${CMAKE_CURRENT_SOURCE_DIR}
  ${CMAKE_CURRENT_SOURCE_DIR}/vmap
  ${CMAKE_CURRENT_SOURCE_DIR}/AuctionHouseBot
  ${CMAKE_CURRENT_SOURCE_DIR}/BattleGround
  ${CMAKE_CURRENT_SOURCE_DIR}/LuaEngine
  ${CMAKE_CURRENT_SOURCE_DIR}/OutdoorPvP
  ${CMAKE_CURRENT_SOURCE_DIR}/PlayerBot
  ${CMAKE_SOURCE_DIR}/dep/lualib
  ${CMAKE_BINARY_DIR}
)

target_include_directories(${LIBRARY_NAME}
  INTERFACE ${CMAKE_CURRENT_SOURCE_DIR}
  PRIVATE ${ADDITIONAL_INCLUDE_DIRS}
  PRIVATE ${Boost_INCLUDE_DIRS}
)

file(GLOB SRC_FILES_LUAENGINE LuaEngine/*.cpp LuaEngine/*.h)
set(SRC_GRP_LUAENGINE
    ${SRC_FILES_LUAENGINE}
)
source_group("LuaEngine"
  FILES
    ${SRC_GRP_LUAENGINE}
)
if(UNIX)
  # Both systems don't have libdl and don't need them
  if (NOT (CMAKE_SYSTEM_NAME STREQUAL "FreeBSD" OR CMAKE_SYSTEM_NAME STREQUAL "NetBSD"))
    target_link_libraries(${LIBRARY_NAME} PUBLIC dl)
  endif()
endif()

# Define BUILD_SCRIPTDEV if need
if (BUILD_SCRIPTDEV)
  add_definitions(-DBUILD_SCRIPTDEV)
endif()

# Define BUILD_PLAYERBOT if need
if (BUILD_PLAYERBOT)
  add_definitions(-DBUILD_PLAYERBOT)
endif()

# Generate precompiled header
if(PCH)
  if (MSVC)
    set(PCHBaseName "pchdef")
    set(PrecompiledBinary "$(IntDir)/${PCHBaseName}.pch")
    set(PrecompiledHeader "${PCHBaseName}.h")
    set(PrecompiledSource "${PCHBaseName}.cpp")
    #set_property(TARGET ${LIBRARY_NAME} PROPERTY COMPILE_FLAGS " /Yu\"${PrecompiledHeader}\" /FI\"${PrecompiledHeader}\" /Fp\"${PrecompiledBinary}\"")
    #set_property(SOURCE ${PrecompiledSource} PROPERTY COMPILE_FLAGS "/Yc\"${PrecompiledHeader}\" /Fp\"${PrecompiledBinary}\"")
    set_target_properties(${LIBRARY_NAME} PROPERTIES COMPILE_FLAGS " /Yu\"${PrecompiledHeader}\" /FI\"${PrecompiledHeader}\" /Fp\"${PrecompiledBinary}\" /bigobj")
    set_source_files_properties(${PrecompiledSource} PROPERTIES COMPILE_FLAGS "/Yc\"${PrecompiledHeader}\" /Fp\"${PrecompiledBinary}\"")
    set_target_properties(${LIBRARY_NAME} PROPERTIES PROJECT_LABEL "Game")
  elseif(NOT MINGW)
    # TODO: Resolve issues with order of includes before enabling Cotire for MinGW builds
    cotire(${LIBRARY_NAME})
  endif()
  include_directories(${CMAKE_CURRENT_BINARY_DIR})
endif()
add_dependencies(${LIBRARY_NAME} lualib)
install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/PlayerBot/playerbot.conf.dist.in DESTINATION ${CONF_DIR} RENAME playerbot.conf.dist)

install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/LuaEngine/extensions DESTINATION ${BIN_DIR}/lua_scripts/)
