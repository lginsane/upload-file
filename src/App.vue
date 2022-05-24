<script setup>
import { ref, reactive, watch, computed } from "vue";
import { ElMessage } from "element-plus";
import axios from "axios";
// 切片大小
const SIZE = 50 * 1024 * 1024;
const HOST = "http://localhost:8088";
// 0: 无状态 1: 上传中 2: 完成 3: 暂停
const status = ref(0);
const container = ref(null);
const fileRef = ref(null);
const fileChunkList = ref([]);
// 取消上传事件
const cancelState = reactive({
  cancel: null,
  cancelLoading: false,
});
const CancelToken = axios.CancelToken;
// 上传进度
const fakeUploadPercentage = ref(0);
// 合并进度
const mergePercentage = ref(0);
// 总进度条包括: 文件上传进度和文件合并进度
const allPercentage = computed(() =>
  parseInt((mergePercentage.value + fakeUploadPercentage.value) / 2)
);
// 生成 hash 进度
const hashPercentage = ref(0);
watch(
  () => fileChunkList.value,
  (val) => {
    if (!fileChunkList.value.length || !container.value.size) return 0;
    const loaded = fileChunkList.value
      .map((item) => item.size * item.percentage)
      .reduce((acc, cur) => acc + cur);
    fakeUploadPercentage.value = parseInt(
      (loaded / container.value.size).toFixed(2)
    );
    // 上传完成，关闭暂停
    if (fakeUploadPercentage.value === 100) {
      cancelState.cancel = null;
    }
  },
  {
    deep: true,
  }
);
// 触发选择文件
function handleUpload() {
  fileRef.value.click();
}
// 选择文件
async function handleFileChange(e) {
  const file = e.target.files[0];
  if (!file) return;
  container.value = file;
  fileChunkList.value = createFileChunk(file);
  const fileHash = await createFileHash(
    fileChunkList.value.map((e) => {
      return { file: e.chunk };
    })
  );
  // 查询已上传切片
  const { shouldUpload, uploadedList } = await verifyUpload(fileHash);
  if (!shouldUpload) {
    initUpload();
    return;
  }
  fileChunkList.value.forEach((item, i) => {
    item.fileHash = fileHash;
    item.hash = fileHash + "-" + i;
    // 过滤已上传切片
    const status =
      uploadedList &&
      uploadedList.length &&
      uploadedList.some((e) => e === item.hash)
        ? 2
        : undefined;
    item.status = status;
    item.percentage = status === 2 ? 100 : 0;
    item.request = function () {
      const formData = new FormData();
      formData.append("chunk", item.chunk);
      formData.append("hash", item.hash);
      formData.append("filename", container.value.name);
      return axios({
        url: HOST + "/file/upload",
        method: "post",
        data: formData,
        onUploadProgress: (e) => {
          item.percentage = parseInt(String((e.loaded / e.total) * 100));
          if (item.percentage === 100) {
            item.status = 2;
          }
        },
        cancelToken: new CancelToken(function executor(c) {
          cancelState.cancel = c;
        }),
      });
    };
  });
  return;
  // 上传中
  status.value = 1;
  await uploadFile();
  await mergeRequest();
  status.value = 2;
  // 上传成功后格式化
  setTimeout(() => {
    initUpload();
  }, 800);
}
// 生成文件切片
function createFileChunk(file, size = SIZE) {
  const fileChunkList = [];
  let currentSize = 0;
  let i = 0;
  while (currentSize < file.size) {
    const bigSize =
      file.size < currentSize + size ? file.size : currentSize + size;
    fileChunkList.push({
      chunk: file.slice(currentSize, currentSize + size),
      // 文件名 + 数组下标
      hash: file.name + "-" + i,
      percentage: 0,
      index: i,
      size: bigSize - currentSize,
    });
    currentSize += size;
    i++;
  }
  return fileChunkList;
}
// 创建文件对应的唯一 hash
function createFileHash(fileChunkList) {
  return new Promise((resolve) => {
    // 添加 worker 属性
    container.value.worker = new Worker("/hash.js");
    container.value.worker.postMessage({ fileChunkList });
    container.value.worker.onmessage = (e) => {
      const { percentage, hash } = e.data;
      hashPercentage.value = parseInt(percentage);
      if (hash) {
        resolve(hash);
      }
    };
  });
}
async function verifyUpload(fileHash) {
  const { data } = await axios({
    url: HOST + "/file/verify",
    method: "post",
    data: {
      fileHash: fileHash,
      filename: container.value.name,
    },
    headers: {
      "content-type": "application/json",
    },
  });
  return data;
}
// 上传文件
async function uploadFile() {
  const requestList = (item) => {
    // 自动跳过上传成功的切片
    if (item.status === 2) {
      return new Promise((resolve) => {
        setTimeout(() => resolve(true), 500);
      });
    }
    return item.request();
  };
  await asyncPool(3, fileChunkList.value, requestList);
}
// 并发数量限制
async function asyncPool(poolLimit, iterable, iteratorFn) {
  const ret = [];
  const executing = new Set();
  for (const item of iterable) {
    const p = Promise.resolve().then(() => iteratorFn(item, iterable));
    ret.push(p);
    executing.add(p);
    const clean = () => executing.delete(p);
    p.then(clean).catch(clean);
    if (executing.size >= poolLimit) {
      await Promise.race(executing);
    }
  }
  return Promise.all(ret);
}
// 合并上传切片
async function mergeRequest() {
  await axios({
    url: HOST + "/file/merge",
    method: "post",
    data: {
      size: SIZE,
      filename: container.value.name,
    },
    headers: {
      "content-type": "application/json",
    },
    // 合并进度条
    onUploadProgress: (e) => {
      mergePercentage.value = parseInt(String((e.loaded / e.total) * 100));
    },
  });
}
// 暂停
async function handlePauseOrContinue() {
  // 处理连续点击
  if (cancelState.cancelLoading) return;
  cancelState.cancelLoading = true;
  if (status.value === 1) {
    if (this.container.worker) {
      this.container.worker.onmessage = null;
    }
    cancelState.cancel && cancelState.cancel();
    status.value = 3;

    ElMessage.warning("已暂停");
    setTimeout(() => (cancelState.cancelLoading = false), 200);
  } else {
    // 上传中
    ElMessage.warning("继续上传");
    status.value = 1;
    await uploadFile();
    await mergeRequest();
    status.value = 2;
    setTimeout(() => (cancelState.cancelLoading = false), 200);
    // 上传成功后格式化
    setTimeout(() => {
      initUpload();
    }, 800);
  }
}
// 格式化
function initUpload() {
  console.log("-----上传成功------格式化-----");
  fileRef.value.value = null;
  fileChunkList.value = [];
  container.value = null;
  fakeUploadPercentage.value = 0;
  mergePercentage.value = 0;
  hashPercentage.value = 0;
  status.value = 0;
  ElMessage.success("上传成功");
}
</script>

<template>
  <div>
    <input
      style="display: none"
      type="file"
      name="file"
      ref="fileRef"
      @change="handleFileChange"
    />
    <el-button type="primary" @click="handleUpload">选择文件上传</el-button>
    <el-button
      type="primary"
      :disabled="status === 0 || !cancelState.cancel"
      @click="handlePauseOrContinue"
      >{{ status === 3 ? "继续" : "暂停" }}
    </el-button>
    <div style="margin-top: 20px">
      <div>生成hash进度条</div>
      <el-progress :percentage="hashPercentage"></el-progress>
    </div>
    <div style="margin-top: 20px">
      <div>文件的进度条</div>
      <el-progress :percentage="allPercentage"></el-progress>
    </div>
    <el-table
      v-if="fileChunkList && fileChunkList.length && hashPercentage === 100"
      :data="fileChunkList"
      style="width: 100%; margin-top: 40px"
    >
      <el-table-column
        prop="hash"
        label="chunk hash"
        align="center"
      ></el-table-column>
      <el-table-column prop="size" label="size(MB)" align="center">
        <template v-slot="{ row }">
          <span>{{ parseInt(row.size / 1024 / 1024) }}</span>
        </template>
      </el-table-column>
      <el-table-column label="percentage" align="center">
        <template v-slot="{ row }">
          <el-progress
            :percentage="row.percentage"
            color="#909399"
          ></el-progress>
        </template>
      </el-table-column>
    </el-table>
  </div>
</template>

<style></style>
