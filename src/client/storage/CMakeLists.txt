target_add_lib(storage-client storage-fbs common mgmtd-client)
find_path(NDS_INCLUDE_DIR nds.h
    PATHS
        /usr/include
        /usr/local/include
        ${CMAKE_SOURCE_DIR}/third_party/CCDK/nds/include
)
if(NDS_INCLUDE_DIR)
    target_include_directories(storage-client PUBLIC ${NDS_INCLUDE_DIR})
endif()
