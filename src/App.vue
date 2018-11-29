<template>
  <div class="app">
    <vue-gic-space
      id="dropzone"
      ref="myDropzone"
      :parentId="parentId"
      :action="folderUrl"
      :options="dropzoneOptions"
      :useCustomSlot="true">
      <i class="el-icon-upload"></i>
      <div class="el-upload__text">将突破或者文件夹拖拽到此处上传，或者点击<em>上传</em>批量上传图片</div>
    </vue-gic-space>
  </div>
</template>

<script>
import VueGicSpace from './components/vue-gic-space';

export default {
  name: 'app',

  components: {
    VueGicSpace
  },

  data() {
    return {
      dropzoneOptions: {
        autoProcessQueue: false,
        url: 'xx', // 上传图片地址
      },
      parentId: 'xxx', // 如果你拖动文件夹里面还有图片 那要传一个id 
      folderUrl: 'xx' // 创建文件夹的地址
    };
  }
};
</script>


