# upload-file

## 大文件上传

### 客户端

- 基于 `Blob.prototype.slice` 方法将文件分割成多个切片
- 基于 `asyncPool` 限制N个并发量，将所有切片记录排序进行上传

### 服务端

- 所有切片上传完成，前端告知后端合并所有切片
- 可以使用 Nodejs 的 读写流（readStream/writeStream），将所有切片的流传输到最终文件的流里
