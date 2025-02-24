<template>
  <el-form>
    <QuillEditor ref="quill-editor" v-model:content="entity.content" :options="editorOption" contentType="html" style="margin-bottom: 10px" />
  </el-form>
</template>
<script>

/*
使用Dufs docker, 见dufs-docker dufs-docker-compose.yaml
依赖
    "quill-image-drop-module": "^1.0.3",
    "quill-image-extend-module": "^1.1.2",
    "quill-image-resize-module--fix-imports-error": "^3.0.0",
    "quill-image-uploader": "^1.3.0",
    "vue-quill-editor": "^3.0.6"

utils/quillTool.js  //处理video的，我们应该用不到
需要配置代理，解决跨域问题
vue.config.js
  server:{
    proxy:{
      '/api':{
        target:"http://127.0.0.1:6990/", //跨域地址
        changeOrigin:true, //支持跨域
        rewrite:(path) => path.replace(/^\/api/, "")//重写路径,替换/api
      }
    }
  }
 */

import {QuillEditor, Quill } from '@vueup/vue-quill'
import { container, ImageExtend, QuillWatch } from 'quill-image-extend-module'
import { ImageDrop } from 'quill-image-drop-module'
import ImageResize from 'quill-image-resize-module--fix-imports-error'

import quillTool from '@/utils/quillTool'
Quill.register(quillTool, true)
Quill.register('modules/ImageExtend', ImageExtend)

Quill.register('modules/imageDrop', ImageDrop)
Quill.register('modules/imageResize', ImageResize)

import 'quill/dist/quill.core.css'
import 'quill/dist/quill.snow.css'
import 'quill/dist/quill.bubble.css'

// 工具栏配置
const toolbarOptions = [
  ['bold', 'italic', 'underline', 'strike'], // 加粗 斜体 下划线 删除线
  ["blockquote", "code-block"], // 引用
  [{ list: 'ordered' }, { list: 'bullet' }], // 有序、无序列表
  [{ script: "sub" }, { script: "super" }], // 上标/下标
  [{ indent: '-1' }, { indent: '+1' }], // 缩进
  [{ direction: 'rtl' }], // 文本方向
  [{ size: ['small', false, 'large', 'huge'] }], // 字体大小
  [{ header: [1, 2, 3, 4, 5, 6, false] }], // 标题
  [{ color: [] }, { background: [] }], // 字体颜色、字体背景颜色
  [{ font: [] }], // 字体种类
  [{ align: [] }], // 对齐方式
  ['clean'], // 清除文本格式
  ['link', 'image', 'video'] // 链接、图片、视频
];
/*富文本编辑图片上传配置*/
const uploadConfig = {
  action: "/api/", // 必填参数 图片上传地址
  methods: 'PUT', // 必填参数 图片上传方式
  // token: "Bearer " + getToken(), // 可选参数 如果需要token验证，假设你的token有存放在sessionStorage
  name: 'file', // 必填参数 文件的参数名
  size: 2000, // 可选参数   图片大小，单位为Kb, 1M = 1024Kb
  accept: 'image/png, image/gif, image/jpeg, image/bmp, image/x-icon' // 可选 可上传的图片格式
};
// image 上传处理
function imageHandler() {
  var self = this;
  var fileInput = this.container.querySelector('input.ql-image[type=file]');
  if (fileInput === null) {
    fileInput = document.createElement('input');
    fileInput.setAttribute('type', 'file');
    // 设置图片参数名
    if (uploadConfig.name) {
      fileInput.setAttribute('name', uploadConfig.name);
    }
    // 可设置上传图片的格式
    fileInput.setAttribute('accept', uploadConfig.accept);
    fileInput.classList.add('ql-image');
    // 监听选择文件
    fileInput.addEventListener('change', function () {
      // 创建formData
      var formData = new FormData();
      var filename = encodeURIComponent(fileInput.files[0].name);
      uploadConfig.action = '/api/'+filename;
      formData.append(uploadConfig.name, fileInput.files[0]);
      // 图片上传
      var xhr = new XMLHttpRequest();
      xhr.open(uploadConfig.methods, uploadConfig.action, true);
      // xhr.setRequestHeader("Authorization",uploadConfig.token);  //设置请求头
      xhr.setRequestHeader('Content-Type', fileInput.files[0].type);
      // 上传数据成功，会触发
      xhr.onload = function (res) {
        if (xhr.status === 201) { // 这里是PUT 201
          // var res = JSON.parse(xhr.responseText);
          let length = self.quill.getSelection(true).index;
          //这里很重要，你图片上传成功后，img的src需要在这里添加，res.path就是你服务器返回的图片链接。
          // self.quill.insertEmbed(length, 'image', res.data);
          self.quill.insertEmbed(length, 'image', 'http://127.0.0.1:6990/' + filename); // TODO后期替换环境变量
          self.quill.setSelection(length + 1)
        }
        fileInput.value = '';
      };
      // 开始上传数据
      xhr.upload.onloadstart = function (e) {
        fileInput.value = ''
      };
      // 当发生网络异常的时候会触发，如果上传数据的过程还未结束
      xhr.upload.onerror = function (e) {
        Message.error({
          message:'图片上传失败'
        })
      };
      // 上传数据完成（成功或者失败）时会触发
      xhr.upload.onloadend = function (e) {
        console.log('上传结束')
      };
      // xhr.send(formData)
      xhr.send(fileInput.files[0]);
    });
    this.container.appendChild(fileInput);
  }
  fileInput.click();
}

export default {
  name: 'EditDictDialog',
  components: { QuillEditor },
  data() {
    return {
      entity:{
        title: null,
        content: null
      },
      submitLoading:false,
      dialogTableVisible:false,
      titleMap:{
        'create':"新增",
        'update':"修改"
      },
      titleStatus:'create',
      rules:{
        title:[
          { required: true,message:'请输入概况标题',  trigger: 'blur' },
          { min: 1, max: 100, message: '长度在100个字符以内', trigger: 'blur' }
        ]
      },
      isDisabledName: false,
      editorOption: {
        theme: 'snow',
        placeholder: '请输入',
        modules: {
          // 处理点击工具栏图片按钮，上传图片base64位转换成服务器图片url
          // ImageExtend: {
          //   loading: true, // 可选参数 是否显示上传进度和提示语
          //   name: 'file_name', // 参数名
          //   action: '', // 服务器地址，如果为空则采用base64插入图片 注意处理跨域
          //   headers: xhr => { // 设置请求头参数（选填）
          //     xhr.setRequestHeader('Content-Type', 'multipart/form-data')
          //   },
          //   response: res => {
          //     console.log(res)
          //     return res.data.imgPath
          //   },
          //   size: 8, // 可选参数 图片大小,图片不能超过8M
          //   sizeError: () => {
          //     this.$message.error('粘贴图片大小不能超过8MB!')
          //   }
          // },
          toolbar: {
            container: toolbarOptions,
            handlers: {
              // 默认处理
              // image: function(value) {
              //   QuillWatch.emit(this.quill.id)
              // },
              image: imageHandler,
              link: function(value) {
                if (value) {
                  var href = prompt('请输入链接地址：')
                  this.quill.format('link', href)
                } else {
                  this.quill.format('link', false)
                }
              },
              video: function(value) {
                if (value) {
                  var href = prompt('请输入视频链接：')
                  this.quill.format('video', href)
                } else {
                  this.quill.format('video', false)
                }
              }
            }
          },
          imageDrop: true,
          imageResize: {
            //放大缩小
            displayStyles: {
              backgroundColor: "black",
              border: "none",
              color: "white"
            },
            modules: ["Resize", "DisplaySize"] //"Toolbar" 无效
          },
        }
      }
    }
  },
  methods: {
    openEditDialog(dialogStatus,titleStatus, row) {
      this.isDisabledName = false
      this.dialogTableVisible=dialogStatus;
      this.titleStatus=titleStatus;
      if(titleStatus==="create"){
        this.cleanData();
      } else if (titleStatus==="update") { // 更新不校验饮片名称是否存在
        this.isDisabledName = true
      }
      if(row){
        this.entity=Object.assign({},row);
        this.$nextTick(() => {
          this.$refs['quill-editor'].setHTML(this.entity.content)
        })
      }
      this.$nextTick(() => {
        this.$refs["form"].clearValidate();
      })

    },
    close(){
      this.dialogTableVisible=false;
      this.cleanData();
      this.titleStatus = 'create'
      this.$nextTick(() => {
        this.$refs['quill-editor'].setHTML('')
        this.$refs['form'].clearValidate()
      })
    }
  }
}
</script>

<style scoped>
:deep() .el-form-item__content {
  display: inline
}
:deep() .el-dialog__footer{
  margin-top: 10px;
}
:deep() .ql-container {
  height: 300px;
  line-height: normal;
  width: auto;
}

:deep() span.ql-size {
  max-width: 80px !important;
}

:deep() .ql-tooltip[data-mode="link"]::before {
  content: "请输入链接地址:";
}

:deep() .ql-tooltip.ql-editing a.ql-action::after {
  border-right: 0px;
  content: "保存";
  padding-right: 0px;
}

:deep() .ql-tooltip[data-mode="video"] {
  left: 0 !important;
}

:deep() .ql-tooltip[data-mode="video"]::before {
  content: "请输入视频地址:";
}

:deep() .ql-picker.ql-size .ql-picker-label::before,
.ql-picker.ql-size .ql-picker-item::before {
  content: "14px";
}

:deep() .ql-picker.ql-size .ql-picker-label[data-value="small"]::before,
.ql-picker.ql-size .ql-picker-item[data-value="small"]::before {
  content: "10px";
}

:deep() .ql-picker.ql-size .ql-picker-label[data-value="large"]::before,
.ql-picker.ql-size .ql-picker-item[data-value="large"]::before {
  content: "18px";
}

:deep() .ql-picker.ql-size .ql-picker-label[data-value="huge"]::before,
.ql-picker.ql-size .ql-picker-item[data-value="huge"]::before {
  content: "32px";
}

:deep() .ql-picker.ql-header .ql-picker-label::before,
.ql-picker.ql-header .ql-picker-item::before {
  content: "文本";
}

:deep() .ql-picker.ql-header .ql-picker-label[data-value="1"]::before,
.ql-picker.ql-header .ql-picker-item[data-value="1"]::before {
  content: "标题1";
}

:deep() .ql-picker.ql-header .ql-picker-label[data-value="2"]::before,
.ql-picker.ql-header .ql-picker-item[data-value="2"]::before {
  content: "标题2";
}

:deep() .ql-picker.ql-header .ql-picker-label[data-value="3"]::before,
.ql-picker.ql-header .ql-picker-item[data-value="3"]::before {
  content: "标题3";
}

:deep() .ql-picker.ql-header .ql-picker-label[data-value="4"]::before,
.ql-picker.ql-header .ql-picker-item[data-value="4"]::before {
  content: "标题4";
}

:deep() .ql-picker.ql-header .ql-picker-label[data-value="5"]::before,
.ql-picker.ql-header .ql-picker-item[data-value="5"]::before {
  content: "标题5";
}

:deep() .ql-picker.ql-header .ql-picker-label[data-value="6"]::before,
.ql-picker.ql-header .ql-picker-item[data-value="6"]::before {
  content: "标题6";
}

:deep() .ql-picker.ql-font .ql-picker-label::before,
.ql-picker.ql-font .ql-picker-item::before {
  content: "标准字体";
}

:deep() .ql-picker.ql-font .ql-picker-label[data-value="serif"]::before,
.ql-picker.ql-font .ql-picker-item[data-value="serif"]::before {
  content: "衬线字体";
}

:deep() .ql-picker.ql-font .ql-picker-label[data-value="monospace"]::before,
.ql-picker.ql-font .ql-picker-item[data-value="monospace"]::before {
  content: "等宽字体";
}
</style>
