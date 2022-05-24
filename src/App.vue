<script setup>
import { ref } from "vue";
import axios from "axios";
// 切片大小
const SIZE = 10 * 1024 * 1024;
const HOST = "http://localhost:8088";

const fileRef = ref(null);
const fileChunkList = ref([]);
// 触发选择文件
function handleUpload() {
  fileRef.value.click();
}
// 选择文件
async function handleFileChange(e) {
  const file = e.target.files[0];
  if (!file) return;
  fileChunkList.value = createFileChunk(file);
  await uploadFile(file.name);
  await mergeRequest(file.name);
}
// 生成文件切片
function createFileChunk(file, size = SIZE) {
  const fileChunkList = [];
  let currentSize = 0;
  let i = 0;
  while (currentSize < file.size) {
    // const bigSize =
    //   file.size < currentSize + size ? file.size : currentSize + size;
    fileChunkList.push({
      chunk: file.slice(currentSize, currentSize + size),
      // 文件名 + 数组下标
      hash: file.name + "-" + i,
      percentage: 0,
      index: i,
      size: SIZE,
    });
    currentSize += size;
    i++;
  }
  return fileChunkList;
}
// 上传文件
async function uploadFile(fileName) {
  const requestList = (item) => {
    const formData = new FormData();
    formData.append("chunk", item.chunk);
    formData.append("hash", item.hash);
    formData.append("filename", fileName);
    return axios({
      url: HOST + "/file/upload",
      method: "post",
      data: formData,
      onUploadProgress: (e) => {
        item.percentage = parseInt(String((e.loaded / e.total) * 100));
      },
    });
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
async function mergeRequest(name) {
  await axios({
    url: HOST + "/file/merge",
    method: "post",
    data: {
      size: SIZE,
      filename: name,
    },
    headers: {
      "content-type": "application/json",
    },
    onUploadProgress: (e) => onMergeProgress(e),
  });
}

// 合并进度条
function onMergeProgress(e) {
  console.log("合并进度", parseInt(String((e.loaded / e.total) * 100)));
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
    <el-button type="primary" @click="handleUpload">上传</el-button>

    <el-table
      :data="fileChunkList"
      style="width: 100%; margin-top: 40px"
      align="center"
    >
      <el-table-column prop="hash" label="chunk hash"></el-table-column>
      <el-table-column prop="size" label="size(KB)"> </el-table-column>
      <el-table-column label="percentage">
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
