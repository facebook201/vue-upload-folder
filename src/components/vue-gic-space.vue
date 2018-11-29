<template>
  <div
    :id="id"
    ref="dropzoneElement"
    :class="{ 'vue-dropzone dropzone': includeStyling }">
    <div
      v-if="useCustomSlot"
      class="dz-message"
      ref="renderMessage">
      <slot>Drop files here to upload</slot>
    </div>
  </div>
</template>

<script>
import Dropzone from 'dropzone';

Dropzone.autoDiscover = false;

export default {
  name: 'vue-upload-folder',

  props: {
    id: {
      type: String,
      required: true,
      default: 'dropzone'
    },
    action: String,
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
    },
    awss3: {
      type: Object,
      default: null
    }
  },

  data() {
    return {
      wasQueueAutoProcess: true,
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
      // 这里我手动限制30张
      this.options.parallelUploads = 30;
      Object.keys(this.options).forEach(function(key) {
        defaultValues[key] = this.options[key]
      }, this);
      return defaultValues
    }
  },

  created() {
    this.folder = [];
    this.folderId = [];
    this.folderName = [];
    this.picName = [];
    // 如是上传单张图片
    this.singlePic = false;
    this.parentIdList = [];
    this.uploadFolder = [];
    this.uploadFlag = false;
  },

  mounted() {
    this.filetypes = ['gif', 'png', 'jpg', 'jpeg'];
    this.limitSize = 1024 * 1024;
    this.removeFlag = false;
    this.hasBeenMounted = true;
    this.dropzone = new Dropzone(this.$refs.dropzoneElement, this.dropzoneSettings);
    let vm = this;

    // 生成缩略图 接收url
    this.dropzone.on('thumbnail', function(file, dataUrl) {
      vm.$emit('vdropzone-thumbnail', file, dataUrl);
    });

    // 将文件添加到上传列表
    this.dropzone.on('addedfile', file => {
      // 如果有fullPath 表示有文件夹
      if (file.fullPath) {
        const path = file.fullPath.split('/');
        this.folder = this.singleFolder(path, 0, this.folder, this.parentId);
        this.deepUpload(this.folder, this.parentId);
      }

      // 限制上传的格式
      let filetype = file.type.split('/')[1];
      if (!this.filetypes.includes(filetype)) {
        this.$message.warning(`您上传的是${filetype}格式，只接收png、jpg、gif的格式`);
        this.removeAllFiles(true);
        return;
      }
      
      // 限制大小
      if (Math.floor(file.size / this.limitSize) >= 5) {
        this.$message.warning(`上传的图片超过5M最大限制，请先压缩再上传！`);
        this.removeAllFiles(true);
        return;
      }
    
      
      setTimeout(() => {
        if (this.getQueuedFiles().length > 30) {
          this.$message.warning(`上传图片总数量不能30张上限！`);
          this.removeAllFiles(true);
        } else {
          this.dropzone.processQueue();
        }
      }, 500);

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

      if (vm.isS3 && vm.wasQueueAutoProcess) {
        vm.getSignedAndUploadToS3(file);
      }
    });

    this.dropzone.on('sending', (file, xhr, formData) => {
      const i = file.fullPath && file.fullPath.split('/').length - 2;
      if (this.singlePic) {
        formData.append('parentId', this.parentId);
      } else {
        formData.append('parentId', this.idList[i]);
      }
    });

    // 将多文件添加到上传列表
    this.dropzone.on('addedfiles', files => {
      // 这里写入第一层的文件夹结构
      if (files.length === 1) {
        if (files[0].name.includes('.')) {
          this.singlePic = true;
        } else {
          // 单个文件夹 我们现在加上
          this.singlePic = false;
        }
      } else {
        // 多个文件
      }
      this.$emit('vdropzone-files-added', files);
    });

    // 移除上传文件
    this.dropzone.on('removedfile', function(file) {
      vm.$emit('vdropzone-removed-file', file);
      if (file.manuallyAddFile) vm.dropzone.options.maxFiles++;
    });

    this.dropzone.on('success', (file, response) => {
      vm.$emit('vdropzone-removed-file', file);
      if (file.manuallyAddFile) vm.dropzone.options.maxFiles++;
    });

    this.dropzone.on('success', (file, response) => {
      // 上传成功之后 删除之前的点
      this.$message.success('上传成功');
      setTimeout(() => {
        // 移除上传成功的图片
        file.previewElement.parentNode.removeChild(file.previewElement);
        if (!document.querySelector('.dz-preview') && this.$refs.renderMessage.style.display !== 'block') {
          this.$refs.renderMessage.style.display = 'block';
        }
      }, 1000);

      vm.$emit('vdropzone-success', file, response);
      if (vm.isS3) {
        if (vm.isS3OverridesServerPropagetion) {
          var xmlResponse = (new window.parseDOM()).parseFormString(response, 'text/xml');
          var s3ObjectLocation = xmlResponse.firstChild.children[0].innerHTML;
          vm.$emit('vdropzone-s3-upload-success', s3ObjectLocation);
        }
        if (vm.wasQueueAutoProcess) {
          vm.setOption('autoProcessQueue', false);
        }
      }
    });

    this.dropzone.on('successmultiple', function(file, response) {
      vm.$emit('vdropzone-success-multiple', file, response);
    });

    // 上传失败的操作
    this.dropzone.on('error', (file, message, xhr) => {
      this.$message.error('上传失败');
      setTimeout(() => {
        // 移除上传成功的图片
        file.previewElement.parentNode.removeChild(file.previewElement);
        if (!document.querySelector('.dz-preview') && this.$refs.renderMessage.style.display !== 'block') {
          this.$refs.renderMessage.style.display = 'block';
        }
      }, 1000);
      this.$emit('vdropzone-error', file, message, xhr);
      if (this.isS3) {
        vm.$emit('vdropzone-s3-upload-error');
      }
    });
   
    this.dropzone.on('complete', file => {
      this.folder.length = 0;
      setTimeout(() => {
        // 移除上传成功的图片
        file.previewElement && file.previewElement.parentNode && file.previewElement.parentNode.removeChild(file.previewElement);
        if (!document.querySelector('.dz-preview') && this.$refs.renderMessage.style.display !== 'block') {
          this.$refs.renderMessage.style.display = 'block';
        }
      }, 1000);
      vm.$emit('vdropzone-complete', file);
    });
   
    this.dropzone.on('canceled', function(file) {
      vm.$emit('vdropzone-canceled', file)
    });

    this.dropzone.on('processing', function(file) {
      vm.$emit('vdropzone-processing', file)
    });

    this.dropzone.on('uploadprogress', function(file, progress, bytesSent) {
      vm.$emit('vdropzone-upload-progress', file, progress, bytesSent)
    });

    // 拖动到框之后
    this.dropzone.on('drop', function(event) {
      vm.$emit('vdropzone-drop', event)
    });

    this.dropzone.on('dragstart', function(event) {
      vm.$emit('vdropzone-drag-start', event)
    });

    this.dropzone.on('dragend', function(event) {
      vm.$emit('vdropzone-drag-end', event)
    });

    this.dropzone.on('dragenter', function(event) {
      vm.$emit('vdropzone-drag-enter', event)
    });

    this.dropzone.on('dragover', function(event) {
      vm.$emit('vdropzone-drag-over', event)
    });

    this.dropzone.on('dragleave', function(event) {
      vm.$emit('vdropzone-drag-leave', event)
    });

    vm.$emit('vdropzone-mounted');
  },

  beforeDestroy() {
    if (this.destroyDropzone) this.dropzone.destroy();
  },

  methods: {
    manuallyAddFile: function(file, fileUrl) {
      file.manuallyAdded = true;
      this.dropzone.emit("addedfile", file);
      let containsImageFileType = false;

      if (fileUrl.indexOf('.png') > -1 || 
          fileUrl.indexOf('.jpg') > -1 ||
          fileUrl.indexOf('.jpeg') > -1) { 
        containsImageFileType = true;
      }
      if (this.dropzone.options.createImageThumbnails &&
          containsImageFileType &&
          file.size <= this.dropzone.options.maxThumbnailFilesize * 1024 * 1024) {
        fileUrl && this.dropzone.emit("thumbnail", file, fileUrl);
        var thumbnails = file.previewElement.querySelectorAll('[data-dz-thumbnail]');
        for (var i = 0; i < thumbnails.length; i++) {
          thumbnails[i].style.width = this.dropzoneSettings.thumbnailWidth + 'px';
          thumbnails[i].style.height = this.dropzoneSettings.thumbnailHeight + 'px';
          thumbnails[i].style['object-fit'] = 'contain';
        }
      }
      this.dropzone.emit("complete", file);
      if (this.dropzone.options.maxFiles) this.dropzone.options.maxFiles--;
      this.dropzone.files.push(file);
      this.$emit('vdropzone-file-added-manually', file);
    },


    // 设置参数
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

   
    disable: function() {
      return this.dropzone.disable();
    },
    
    filesize: function(size) {
      return this.dropzone.filesize(size);
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
    
    getAcceptedFiles: function() {
      return this.dropzone.getAcceptedFiles()
    },
    
    getRejectedFiles: function() {
      return this.dropzone.getRejectedFiles()
    },
    
    getFilesWithStatus: function() {
      return this.dropzone.getFilesWithStatus()
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
    getActiveFiles: function() {
      return this.dropzone.getActiveFiles()
    },

    // 一层的情况
    singleFolder(newPath, level, folder, parentId) {
      if (!newPath[level].includes('.') ) {
        if (folder.length) {
          folder.forEach(item => {
            if (item.name != newPath[level]) {
              const node = {
                parentId: parentId,
                children: [],
                isDir: true,
                name: newPath[level],
                level
              };

              folder.push(node);
              if (newPath[level + 1]) {
                const children = this.singleFolder(newPath, level + 1, node.children);
                node.children = children;
              }
            } else if(newPath[level + 1]) {
              const children = this.singleFolder(newPath, level + 1, item.children);
              item.children = children;
            }
          })
        } else {
          const node = {
            parentId: parentId,
            children: [],
            isDir: true,
            name: newPath[level],
            level
          };
          if (name.name != newPath[level]) {
            folder.push(node);
            if (newPath[level + 1]) {
              const children = this.singleFolder(newPath, level + 1, node.children);
              node.children = children;
            }
          }
        }
      } else {
        if (folder.length) {
          folder.forEach(item => {
            if (item.name != newPath[level]) {
              const node = {
                isDir: false,
                name: newPath[level],
                level
              };
              folder.push(node);
            }
          })
        } else {
          const node = {
            isDir: false,
            name: newPath[level],
            level
          };
          folder.push(node);
        }
      }
      return folder;
    },


    // 遍历第一层 递归后面的路径参数
    addChildren(folder, path) {
      folder.forEach(item => {
        if (path[0] == item.name) {
          const node = {
            ...item,
            level: 0
          };
          folder.push(node);
          if (node.children) {
            const children = this.addDeepFolder(path, node.level);
            node.children = children;
          }
        }
      });
      return folder;
    },

    // 递归添加文件夹和图片
    addDeepFolder(path, level) {
      const nodes = [];
      if (path[level + 1]) {
        if (!path[level + 1].includes('.')) {
          const node = {
            level: level + 1,
            name: path[level + 1],
            parentId: null,
            children: [],
            isDir: true
          };
          nodes.push(node);
          if (node.children) {
            const children = this.addDeepFolder(path, level + 1);
            node.children = children;
          }
        } else {
            const node = {
              level: level + 1,
              name: path[level + 1],
              parentId: null,
              children: null,
              isDir: false
            };
            nodes.push(node);
        }
      }
      return nodes;
    },

    deepUpload(folder, returnParentId) {
      // 如果存在文件夹
      folder.forEach(item => {
        if (!item.name.includes('.')) {
          if (!this.uploadFolder.includes(item.name)) {
            this.uploadFolder.push(item.name);
            const id = item.parentId ? item.parentId : returnParentId;
            // 不存在就开始上传
            this.axios.get(`
              xxx
              `).then(res => {
                if (res.data) {
                  if (res.data.errorCode === 0) {
                    const returnParentId = res.data.result.id;
                    this.idList.push(returnParentId);
                    if (item.children && item.children.length) {
                      setTimeout(() => {
                        this.deepUpload(item.children, returnParentId);
                      }, 10);
                    }
                  } else if (res.data.errorCode === 1) {
                    this.$message.error('已存在改文件夹！');
                    this.removeAllFiles(true);
                    return;
                  }
                }
              });
          }
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