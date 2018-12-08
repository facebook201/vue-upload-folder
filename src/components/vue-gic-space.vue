<template>
  <div
    :id="id"
    ref="dropzoneElement"
    :class="{ 'vue-dropzone dropzone': includeStyling }">
    <div
      v-if="useCustomSlot"
      class="dz-message"
      ref="renderMessage">
      <slot>拖拽上传文件夹</slot>
    </div>
  </div>
</template>

<script>
import Dropzone from 'dropzone';

Dropzone.autoDiscover = false;

export default {
  name: 'upload-folder-space',

  props: {
    // 组件定义的id值 默认 dropzone
    id: {
      type: String,
      required: true,
      default: 'dropzone'
    },
    action: String,
    requestProject: String, 
    parentId: String,
    options: {
      type: Object,
    },
    includeStyling: {
      type: Boolean,
      default: true
    },
    requestParams: {
      type: Object,
      default: null
    },
    // 销毁dropzone
    destroyDropzone: {
      type: Boolean,
      default: true
    },
    useCustomSlot: {
      type: Boolean,
      default: false
    }
  },

  data() {
    return {
      idList: []
    };
  },

  computed: {
    dropzoneSettings() {
      // 设置默认宽高
    let defaultValues = {
        thumbnailWidth: 200,
        thumbnailHeight: 200
      };
      // 限制上限 1000
      this.options.parallelUploads = 1000;
      Object.keys(this.options).forEach(function(key) {
        defaultValues[key] = this.options[key]
      }, this);
      return defaultValues
    }
  },

  created() {
    this.folder = [];
    // 如是上传单张图片
    this.uploadFolder = [];
    this.uploadFlag = false;
    this.requestUrl = window.location.origin;
    // this.requestUrl = this.requestUrl; 这里自己修改自己对应的接口地址
  },

  mounted() {
    this.filetypes = ['gif', 'png', 'jpg', 'jpeg'];
    this.limitSize = 1024 * 1024;
    this.removeFlag = false;
    this.hasBeenMounted = true;
    this.dropzone = new Dropzone(this.$refs.dropzoneElement, this.dropzoneSettings);
    let vm = this;

    // 将文件添加到上传列表
    this.dropzone.on('addedfile', file => {
      // 如果有fullPath 表示有文件夹
      if (file.fullPath) {
        const path = file.fullPath.split('/');
        this.folder = this.singleFolder(path, 0, this.folder);
      }
      // 限制上传的格式
      let filetype = file.type.split('/')[1];
      if (!this.filetypes.includes(filetype)) {
        this.$message.warning(`您上传的是${filetype}格式，只接收png、jpg、gif的格式`);
        this.removeFile(file);
        return;
      }
      
      // 限制大小
      if (Math.floor(file.size / this.limitSize) >= 5) {
        this.$message.warning(`上传的图片超过5M最大限制，请先压缩再上传！`);
        this.removeFile(file);
        return;
      }

      if (!this.uploadFlag) {
        // 先把树形结构组装起来
        this.uploadFlag = true;
        setTimeout(() => {
          this.deepUpload(this.folder, this.parentId);
        }, 100);
        setTimeout(_ => {
          console.log(this.folder);
          this.dropzone.processQueue();
        }, 1500);
      }

      if (this.$refs.renderMessage.style.display === 'block') {
        this.$refs.renderMessage.style.display = 'none';
      }
      var isDuplicate = false;
      if (vm.duplicateCheck) {
        if (this.files.length) {
          var $i, $len;

          for ($i = 0, $len = this.files.length; $i < $len - 1; $i++) {
            if (this.files[$i].name === file.name &&
              this.files[$i].size === file.size &&
              this.files[$i].lastModifiedDate.toString() === file.lastModifiedDate.toString()) {
              this.removeFile(file);
              isDuplicate = true;
              vm.$emit('vdropzone-duplicate-file', file);
            }
          }
        }
      }
      
      vm.$emit('vdropzone-file-added', file);
    });

    this.dropzone.on('sending', (file, xhr, formData) => {
      // 找id根据名字
      if (file.fullPath) {
        const pathArr = file.fullPath.split('/');
        const LEN = pathArr.length - 2;
        const id = this.findId(this.folder, pathArr, 0, LEN);
        formData.append('parentId', id);
      } else {
        formData.append('parentId', this.parentId);
      }
    });

    // 将多文件添加到上传列表
    this.dropzone.on('addedfiles', files => {
      // 如果是拖动上传文件夹和图片 或者多个图片 
      this.$emit('files-added', files);
    });

    // 移除上传文件
    this.dropzone.on('removedfile', function(file) {
      vm.$emit('removed-file', file);
      if (file.manuallyAddFile) vm.dropzone.options.maxFiles++;
    });

    this.dropzone.on('success', (file, response) => {
      this.$message.success('上传成功');
      setTimeout(() => {
        file.previewElement.parentNode.removeChild(file.previewElement);
        if (!document.querySelector('.dz-preview') && this.$refs.renderMessage.style.display !== 'block') {
          this.$refs.renderMessage.style.display = 'block';
        }
      }, 100);

      vm.$emit('upload-success', file, response);
    });

   

    this.dropzone.on('error', (file, message, xhr) => {
      this.$message.error('上传失败');
      setTimeout(() => {
        file.previewElement.parentNode.removeChild(file.previewElement);
        if (!document.querySelector('.dz-preview') && this.$refs.renderMessage.style.display !== 'block') {
          this.$refs.renderMessage.style.display = 'block';
        }
      }, 100);
    });

    
    this.dropzone.on('complete', file => {
      this.uploadFlag = false;
      this.folder.length = 0;
      setTimeout(() => {
        // 移除上传成功的图片
        file.previewElement && file.previewElement.parentNode && file.previewElement.parentNode.removeChild(file.previewElement);
        if (!document.querySelector('.dz-preview') && this.$refs.renderMessage.style.display !== 'block') {
          this.$refs.renderMessage.style.display = 'block';
        }
        vm.$emit('upload-complete');
      }, 100);
      
    });

   
    this.dropzone.on('processing', function(file) {
      vm.$emit('upload-processing', file)
    });

    this.dropzone.on('uploadprogress', function(file, progress, bytesSent) {
      vm.$emit('upload-progress', file, progress, bytesSent)
    });

    this.dropzone.on('queuecomplete', function(file) {
      vm.$emit('queue-complete')
    });

    // 拖动到框之后
    this.dropzone.on('drop', function(event) {
      vm.$emit('upload-drop', event)
    });

    vm.$emit('upload-mounted');
  },

  beforeDestroy() {
    if (this.destroyDropzone) this.dropzone.destroy();
  },

  methods: {
    setOption: function(option, value) {
      this.dropzone.options[option] = value
    },
    removeAllFiles: function(bool) {
      this.dropzone.removeAllFiles(bool)
    },
    processQueue: function() {
      let dropzoneEle = this.dropzone;
      if (this.isS3 && !this.wasQueueAutoProcess) {
        this.getQueuedFiles().forEach((file) => {
          this.getSignedAndUploadToS3(file);
        });
      } else {
        this.dropzone.processQueue();
      }
      this.dropzone.on("success", function() {
        dropzoneEle.options.autoProcessQueue = true;
      });

      this.dropzone.on('queuecomplete', function() {
        dropzoneEle.options.autoProcessQueue = false;
      });
    },
    // 初始化
    init: function() {
      return this.dropzone.init();
    },
    // 销毁实例
    destroy: function() {
      return this.dropzone.destroy();
    },
    accept: function(file, done) {
      return this.dropzone.accept(file, done);
    },
    addFile: function(file) {
      return this.dropzone.addFile(file);
    },
    removeFile: function(file) {
      this.dropzone.removeFile(file)
    },
    getQueuedFiles: function() {
      return this.dropzone.getQueuedFiles()
    },
    getUploadingFiles: function() {
      return this.dropzone.getUploadingFiles()
    },
    getAddedFiles: function() {
      return this.dropzone.getAddedFiles()
    },
    /**
     * @param { newPath 文件路径拆开的数组 }
     * @param { level 层级 表示是第几层 }
     * @param { folder 传进来的文件夹树形结构 首次为空数组 }
     */
    singleFolder(newPath, level, folder) {
      let sum = 0;
      const ret = JSON.parse(JSON.stringify(folder));
      // newPath[level] 不能跟folder名字相同
      if (!newPath[level].includes('.')) {
        // 如果不是空文件夹 
        if (ret.length) {
          // 如果里面有就添加到当前下面 没有就添加到这一层
          for (let i = 0; i < ret.length; i++) {
            if (newPath[level] == folder[i].name) {
              if (newPath[level + 1]) {
                ret[i].children = this.singleFolder(newPath, level + 1, ret[i].children);
                return ret;
              }
            } else {
              sum++;
            }
          }
          // 如果能走到这里就表示不相等
          if (sum == ret.length) {
            const node = {
              children: [],
              isDir: true,
              name: newPath[level],
              level
            };
            ret.push(node);
            if (newPath[level + 1]) {
              node.children = this.singleFolder(newPath, level + 1, node.children);
            }
          }
        } else {
          const node = {
            children: [],
            isDir: true,
            name: newPath[level],
            level
          };
          ret.push(node);
          // 看后面还有没有元素
          if (newPath[level + 1]) {
            node.children = this.singleFolder(newPath, level + 1, node.children);
          }
        }
      } else {
          const node = {
            isDir: false,
            name: newPath[level],
            level
          };
          ret.push(node);
      }
      return ret;
    },

    /** 查找id
     * @param list 树形结构
     * @param arr 文件的路径
     * @param 文件的层级
     * @param 路径的length
     * return 返回的是 这个图片是哪个文件夹下对应的id
     */
    findId(list, arr, level, len) {
      let id;
      let i = 0;
      while(i < list.length) {
        if (list[i].name == arr[level]) {
          if (level == len) {
            id = list[i].parentId;
            return id;
          }
          if (arr[level + 1] && !arr[level + 1].includes('.')) {
            id = this.findId(list[i].children, arr, level + 1, len);
          }
        }
        i++;
      }
      return id;
    },
    //上传文件夹
    deepUpload(folder, returnParentId) {
      // 如果存在文件夹
      folder.forEach(item => {
        if (item.isDir) {
          this.uploadFolder[item.level] = this.uploadFolder[item.level] || [];
            const id = item.parentId ? item.parentId : returnParentId;
            // 不存在就开始上传
            this.axios.get(`
              xxx
              `).then(res => {
                if (res.data) {
                  if (res.data.errorCode === 0) {
                  // 成功之后返回的 parentId 添加到 
                    const returnParentId = res.data.result.id;
                    item.parentId = returnParentId;
                    if (item.children && item.children.length) {
                      setTimeout(() => {
                        this.deepUpload(item.children, returnParentId);
                      }, 0);
                    }
                  } else if (res.data.errorCode === 1) {
                    this.$message.error('已存在该文件夹！');
                    this.removeAllFiles(true);
                    return;
                  }
                }
              });
          }
      });
    }
  }
};
</script>

 <style lang="less">
  @import (inline) '../../node_modules/dropzone/dist/dropzone.css';

  .vue-dropzone {
    width: 500px;
    height: 225px;
    overflow: hidden;
    border: 1px dashed #E5E5E5;
    font-family: 'Arial', sans-serif;
    letter-spacing: 0.2px;
    color: #777;
    transition: background-color .2s linear;
    &:hover {
      background-color: #F6F6F6;
    }
    i {
      color: #CCC;
    }
    .dz-preview {
      .dz-image {
        img:not([src]) {
          width: 200px;
          height: 200px;
        }
        &:hover {
          img {
            transform: none;
            -webkit-filter: none;
          }
        }
      }
      .dz-details {
        bottom: 0;
        top: 0;
        color: white;
        background-color: rgba(33, 150, 243, 0.8);
        transition: opacity .2s linear;
        text-align: left;
        .dz-filename {
          overflow: hidden;
        }
        .dz-filename span,
        .dz-size span {
          background-color: transparent;
        }
        .dz-filename:not(:hover) span {
          border: none;
        }
        .dz-filename:hover span {
          background-color: transparent;
          border: none;
        }
      }
      .dz-progress .dz-upload {
        background: #4094ff;
      }
      .dz-remove {
        position: absolute;
        z-index: 30;
        color: white;
        margin-left: 15px;
        padding: 10px;
        top: inherit;
        bottom: 15px;
        border: 2px white solid;
        text-decoration: none;
        text-transform: uppercase;
        font-size: 0.8rem;
        font-weight: 800;
        letter-spacing: 1.1px;
        opacity: 0;
      }
      &:hover {
        .dz-remove {
          opacity: 1;
        }
      }
      .dz-success-mark,
      .dz-error-mark {
        margin-left: auto;
        margin-top: auto;
        width: 100%;
        top: 20%;
        left: 0;
        svg {
          margin-left: auto;
          margin-right: auto;
        }
      }
      .dz-error-message {
        top: calc(15%);
        margin-left: auto;
        margin-right: auto;
        left: 0;
        width: 100%;
        &:after {
          bottom: -6px;
          top: initial;
          border-top: 6px solid #a92222;
          border-bottom: none;
        }
      }
    }
    .el-icon-upload {
      font-size: 57px;
    }
    b {
      color: #4094ff;
    }
  }
</style>