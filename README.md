## vue-upload-folder

#### 安装

```
npm i vue-upload-space
```



#### 使用

```javascript
import Vue from 'vue';
import VueUploadSpace from 'vue-upload-space';

Vue.use(VueUploadSpace);
```




```vue
    <vue-gic-space
      id="dropzone"
      ref="myDropzone"
      :options="dropzoneOptions"
      :action="folderUrl"
      :parentId="parentId"
      :useCustomSlot="true">
      <i class="el-icon-upload"></i>
      <div class="el-upload__text">将图片或文件夹拖到此处上传，或点击<b>上传</b>图片</div>
      <div class="upload-tips">仅支持5M以内的jpg、png、gif格式的图片</div>
    </vue-gic-space>
    

<script>
export default {

  data() {
    return {
      // 传给dropzone的配置项
      dropzoneOptions: {
        autoProcessQueue: false, // 不立即上传
        url: 'xx', // 上传图片地址
      },
      parentId: 'xxx', // 如果你拖动文件夹里面还有图片 那要传一个id 
      folderUrl: 'xx' // 创建文件夹的地址
    };
  }
};
</script>
```



### Documentation 更多参数可看dropzone的文档

> props

此组件基于 [dropzone](http://dropzonejs.com "dropzone") 库实现的。 相关配置参数可看官网

| prop name       | type    | default  | description                                                 | required |
| --------------- | ------- | -------- | ----------------------------------------------------------- | -------- |
| id              | string  | dropzone | 组件定义的id名                                              | 必传     |
| options         | object  | { }      | dropzone 配置对象 可看dropzone官网                          | 必传     |
| destoryDropzone | boolean | true     | 组件销毁时会销毁dropzone实例                                | 不是必传 |
| useCustomeSlot  | boolean | false    | 用户自定义的分发内容  （dropzone库有默认 一般我们自己添加） | 不是必传 |
| folderUrl       | String  | ''       | 文件夹上传地址（如果有文件夹要上传）                        | 选传     |



> events

| Event Name                       | Description                                                  |
| -------------------------------- | ------------------------------------------------------------ |
| file-added(file)                 | 添加文件到dropzone                                           |
| upload-success(file, response)   | 文件上传成功 获取服务端返回响应                              |
| upload-complete(file)            | 上传完成之后被调用 无论上传成功或失败                        |
| upload-error(file, message, xhr) | 上传失败后被调用                                             |
| upload-progress(file, progress)  | 文件上传进度 上传发生进度发生变化 都会定期调用 获取progress参数  至少调用一次 |



> methods


| 方法                | 描述                                                         |
| ------------------- | ------------------------------------------------------------ |
| removeAllFiles()    | 移除所有文件 正在上传的文件不会删除 如果要取消在上传的文件 翻看dropzone |
| removeFile(file)    | 移除dropzone区域的文件                                       |
| getUploadingFiles() | 获取所有上传的文件                                           |
| getQueuedFiles()    | 获取所有队列的文件                                           |
| getRejectedFiles()  | 失败的文件                                                   |
| getAcceptedFiles()  | 成功的文件                                                   |

```javascript

methods: {
	// 调用方法
	this.$refs.myDropzone.xxxMethod();
}


```