if(CMAKE_SYSTEM_PROCESSOR STREQUAL "x86_64")
    link_directories(/usr/local/lib/x86_64-linux-gnu/ /usr/lib64 /usr/local/lib64)
elseif(CMAKE_SYSTEM_PROCESSOR STREQUAL "aarch64")
    link_directories(/usr/local/lib/aarch64-linux-gnu/ /usr/lib64 /usr/local/lib64)
endif()

target_add_lib(hf3fs_fuse common core-app meta-client storage-client fuse3 client-lib-common)
find_path(NDS_INCLUDE_DIR nds.h
    PATHS
        /usr/include
        /usr/local/include
        ${CMAKE_SOURCE_DIR}/third_party/CCDK/nds/include
)
if(NDS_INCLUDE_DIR)
    target_include_directories(hf3fs_fuse PUBLIC ${NDS_INCLUDE_DIR})
endif()
target_add_bin(hf3fs_fuse_main hf3fs_fuse.cpp hf3fs_fuse)

if (ENABLE_FUSE_APPLICATION)
target_compile_definitions(hf3fs_fuse PUBLIC -DENABLE_FUSE_APPLICATION)
endif()
