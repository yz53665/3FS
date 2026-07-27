add_subdirectory(cli)
add_subdirectory(bin)
add_subdirectory(mgmtd)
add_subdirectory(meta)
add_subdirectory(storage)
add_subdirectory(core)

# Find NDS library (system or third_party)
find_path(NDS_INCLUDE_DIR nds.h
    PATHS
        /usr/include
        /usr/local/include
        ${CMAKE_SOURCE_DIR}/third_party/CCDK/nds/include
)
find_library(NDS_LIBRARY nds
    PATHS
        /usr/lib
        /usr/lib64
        /usr/local/lib
        /usr/local/lib64
        ${CMAKE_SOURCE_DIR}/third_party/CCDK/output/lib
)
if(NDS_INCLUDE_DIR)
    target_include_directories(storage PUBLIC ${NDS_INCLUDE_DIR})
    message(STATUS "NDS include found: ${NDS_INCLUDE_DIR}")
else()
    message(WARNING "NDS header not found, NPU direct IO will not compile")
endif()

if(NDS_LIBRARY)
    target_link_libraries(storage_main PRIVATE ${NDS_LIBRARY})
    message(STATUS "NDS library found: ${NDS_LIBRARY}")
endif()