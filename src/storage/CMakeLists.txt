add_crate(chunk_engine)

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
)

target_add_lib(storage core-app core-service memory-common storage-fbs mgmtd-client storage-client kv analytics aio chunk_engine profiler)
target_include_directories(storage PUBLIC ${CMAKE_SOURCE_DIR}/third_party/leveldb)
if(NDS_INCLUDE_DIR)
    target_include_directories(storage PUBLIC ${NDS_INCLUDE_DIR})
    message(STATUS "NDS include found: ${NDS_INCLUDE_DIR}")
else()
    message(WARNING "NDS header not found, NPU direct IO will not compile")
endif()
target_add_bin(storage_main "storage.cpp" storage jemalloc)
if(NDS_LIBRARY)
    target_link_libraries(storage_main PRIVATE ${NDS_LIBRARY})
    message(STATUS "NDS library found: ${NDS_LIBRARY}")
endif()
