target_add_lib(hf3fs_api client-lib-common storage-client numa rt)
target_add_shared_lib(hf3fs_api_shared client-lib-common storage-client numa rt)
find_path(NDS_INCLUDE_DIR nds.h
    PATHS
        /usr/include
        /usr/local/include
        ${CMAKE_SOURCE_DIR}/third_party/CCDK/nds/include
)
if(NDS_INCLUDE_DIR)
    target_include_directories(hf3fs_api PUBLIC ${NDS_INCLUDE_DIR})
    target_include_directories(hf3fs_api_shared PUBLIC ${NDS_INCLUDE_DIR})
endif()

add_custom_command(
    TARGET hf3fs_api_shared
    POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E create_symlink "${CMAKE_CURRENT_BINARY_DIR}/libhf3fs_api_shared.so" "${CMAKE_SOURCE_DIR}/src/lib/rs/hf3fs-usrbio-sys/lib/libhf3fs_api_shared.so"
    COMMENT "linking usrbio library to rust binding lib dir"
)
