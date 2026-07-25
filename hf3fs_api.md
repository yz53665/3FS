```
int hf3fs_prep_npu_direct_io(const struct hf3fs_ior *ior
	bool read,
    int fd,
    size_t off,
    uint64_t len,
    nds_segment_info_t *nds_segment_info,
    void *nds_buf_addr,  // NPU HBM物理地址
    uint64_t nds_buf_size,
    const void *userdata);
```
