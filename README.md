# upload-file

## 大文件上传

### 客户端

- 基于 `Blob.prototype.slice` 方法将文件分割成多个切片
- 基于 `asyncPool` 限制N个并发量，并将所有切片`记录排序`进行上传
- 所有切片上传完成，前端通知后端合并所有切片

### 服务端

- 基于 Nodejs 的 读写流`readStream/writeStream`，将所有切片的流传输到最终文件的流里

## 断点续传

### 客户端

- 基于`spark-md5`生成文件对应的唯一`hash`, 用来处理上传暂停后，跳过已上传文件
  - 利用fileRender读取每个切片的ArrayBuffer，将其传入`spark-md5`中
  - 基于`worker`线程计算`hash`, 从而不影响主界面正常交互
- 基于`axios`中`abort()`方法取消上传

### 服务端

- 根据唯一`hash`, 查询已上传切片或者文件

