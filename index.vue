<template>
    <div :class="['uploader-box', boxClass]">
        <input style="display: none;" type="file" name="file" 
        :id="'uploader-' + id" 
        :accept="accept"
        :multiple="multiple"
        :disabled="disabled"
        :preview="preview"
        @change="afterSelect"
        ></input>
        <img v-if="isShowBefore==true" v-bind:src="isShowUrl" alt="" :class="imgClass"  @click="selectFilesFn">
        <el-button v-else plain :class="areaClass"  @click="selectFilesFn">点击上传</el-button>
    </div>
</template>

<script>
import { Message } from 'element-ui'
import axios from 'axios'

export default {
    name: 'Upload',
    // 组合其它组件
    extends: {},
    // 组件属性、变量
    props: {
        boxClass: String,//上传box添加class类
        areaClass:String,//上传按钮class类
        //限制文件上传类型
        accept:{
            type: String,
            default(){
                return 'image/*'
            }
        },
        imgClass:String,
        //上传之前是否要显示默认图
        isShowBefore:{
            type:Boolean,
            default:false
        },
        //上传之前要显示默认图的url
        isShowUrl:String,
        //是否支持多文件上传
        multiple:{
            type:Boolean,
            default:false
        },
        //是否支持预览
        preview:{
            type:Boolean,
            default:false
        },
        //最大允许上传个数
        limit:{
            type:Number,
            default:1
        },
        //文件大小限制(单位：kb)
        sizeLimit:{
            type:Number,
            default:102400
        },
        //是否设置disabled（禁用上传）
        disabled:{
            type:Boolean,
            default:false
        }
    },
    // 变量
    data() {
        return {
            //上传input 随机id
            id: Math.random().toString(36).substr(5),
            //oss上传阿里云主体（document.getElementById("uploader-"+this.id).files[0];）
            // uploadBody:null,
            form:{
                fileList:{},//选择的上传文件filelist对象
                storeList:[],//返回的数据（oldName,newName,url,size）
                name:''
            },
            host:'',//上传路径
            expire:0,//上传秘钥有效时间戳
            key:'',//上传路径 + 文件名（后缀）
            isUpload:false,
            //oss 上传秘钥
            uploadOpts:{
                policy:'',
                OSSAccessKeyId:'',
                success_action_status:200,
                signature:''
            }
        }
    },
    // 生命周期函数
    mounted() {
        // console.log(this.disabled)
        // console.log(this.boxClass)
    },
    watch: {
        
    },
    methods: {
        //触发 input[type="file"]的点击事件
        selectFilesFn(){
            let fileInput = document.getElementById("uploader-"+this.id);
            //先获取一遍oss上传秘钥
            this.getSignData();
            //触发点击
            fileInput.click();
        },
        //选择文件后 先获取选择的文件
        afterSelect(e){
            let fileInput = document.getElementById("uploader-"+this.id);
            let files = fileInput.files;
            //文件数量超出限制
            if(!this.checkFilesCount(files.length)){
                return false;
            };
            //文件大小验证
            if(!this.checkFilesSize(files)){
                return false;
            };
            //通知父组件 即将开始上传(也可看作是选择文件后的事件，无法打断上传)
            this.beforeUpload(files);
            //上传预览
            this.previewFn(files);
            //设置上传list name
            this.setUploadList(files);
            //验证oss上传秘钥是否有效 并开始上传
            this.checkExpire();
        },
        //判断上传文件数量是否符合
        checkFilesCount(count){
            if(this.multiple && count>this.limit){
                Message({
                    message: "请选择少于"+this.limit+"个文件上传。",
                    type: 'error',
                    duration: 3 * 1000
                });
                return false;
            }else if(count <= 0){
                return false;
            }else{
                return true;
            };
        },
        //文件大小验证
        checkFilesSize(files){
            for (let index in files) { 
                if(typeof files[index] === 'object'){
                    let _size = files[index].size;
                    if(_size>this.sizeLimit*1024){
                        Message({
                            message: "请选择小于"+this.sizeLimit+"Kb的文件上传。",
                            type: 'error',
                            duration: 3 * 1000
                        });
                        return false;
                    };
                }
            };
            return true;
        },
        //上传预览方法
        previewFn(files){
            let self = this;
            if(!self.preview){
                return;
            };
            // 看支持不支持FileReader
            if(!window.FileReader){
                Message({
                    message: "你的浏览器不支持预览，请升级或更换浏览器。",
                    type: 'error',
                    duration: 3 * 1000
                });
                return;
            };
            const _list = [];
            //循环files
            for(let i in files){
                if(typeof files[i] === 'object'){
                    //如果是图片
                    if(/^image/.test(files[i].type)){
                        // 创建一个reader
                        var reader = new FileReader();
                        // 将图片将转成 base64 格式
                        reader.readAsDataURL(files[0]);
                        // 读取成功后的回调
                        reader.onloadend = function () {
                            let _file = {
                                type:'image',
                                name:files[0].name,
                                url:this.result,
                                size:files[0].size
                            };
                            _list.push(_file)
                        }
                    }else{//如果不是图片，则不返回url
                        let _file = {
                            type:files[0].type,
                            name:files[0].name,
                            url:'',
                            size:files[0].size
                        };
                        _list.push(_file)
                    }
                }
            };
            self.onPreview(_list);
        },
        //设置上传list name
        setUploadList(list){
            this.form.fileList = list;
            // this.form.list = [];
            this.form.storeList = [];
            for (let index in list) { 
                if(typeof list[index] === 'object'){
                    // console.log('正在上传第'+index+"个文件")
                    let fileName = this.getUploadName(list[index].name);
                    this.form.storeList.push({
                        oldName:list[index].name,//上传前文件名
                        newName:fileName,//上传后文件名
                        url:this.host+"/"+this.key+fileName,//上传地址
                        size:list[index].size//文件大小（字节Byte）
                    });
                }
            };
        },
        
        //获取oss上传秘钥
        getSignData(){
            let self = this;
            axios.post("http://api.com/sign.json").then(function(response) {  
                let res = response.data.data;
                self.host = res.host;
                self.key = res.dir;
                self.uploadOpts.policy = res.policy;
                self.uploadOpts.OSSAccessKeyId = res.accessid;
                self.uploadOpts.signature = res.signature;
                self.expire = parseInt(res.expire);
                if(self.isUpload){
                    self.uploadFn();
                };
            }).catch(function(error) {  
                console.log('获取上传秘钥失败。');
                Message({
                    message: error,
                    type: 'error',
                    duration: 3 * 1000
                });  
            })
        },
        //设置上传文件名
        getUploadName(name){
            let suffix = this.get_suffix(name)
            let uploadName = this.random_string(10) + suffix
            return uploadName;
        },
        //uid 获取随机排布的len位字符串
        random_string(len) {
            len = len || 32;
            let chars = 'ABCDEFGHJKMNPQRSTWXYZabcdefhijkmnprstwxyz2345678';
            let maxPos = chars.length;
            let pwd = '';
            for (let i = 0; i < len; i++) {
                pwd += chars.charAt(Math.floor(Math.random() * maxPos));
            }
            return pwd;
        },
        //获取后缀名
        get_suffix(filename){
            let pos = filename.lastIndexOf('.')
            let suffix = ''
            if (pos != -1) {
                suffix = filename.substring(pos)
            };
            return suffix;
        },
        //验证 oss上传秘钥是否有效
        checkExpire(){
            let _time = parseInt((new Date().getTime())/1000);
            if(this.expire>0 && this.expire >= _time+3){
                console.log('秘钥有效，可以上传')
                this.beginUpload();
            }else{
                this.isUpload = false;
                console.log('秘钥失效，重新获取token')
                this.getSignData();
            };
        },
        // 开始上传
        beginUpload(){
            let self = this;
            //上传变量
            for (let index in self.form.fileList) {
                if(typeof self.form.fileList[index] === 'object'){
                    console.log('正在上传第'+index+"个文件");
                    self.uploadFn(index,self.buildFormData(index));
                }
            };
        },
        //重新设置formdata
        buildFormData(index){
            let formData = new FormData();
            //上传的固定参数
            for (let i in this.uploadOpts) {
                formData.append(i,this.uploadOpts[i]);
            };

            let _file = this.form.storeList[index];
            if(formData.has('key')){
                formData.set('name',_file.oldName);
                formData.set('key', this.key + _file.newName);
                formData.set('file',this.form.fileList[index]);
            }else{
                formData.append('name',_file.oldName);
                formData.append('key', this.key + _file.newName);
                formData.append('file',this.form.fileList[index]);
            };
            return formData;
        },
        //上传方法
        uploadFn(index,formData){
            let self = this;
            let config = {
                headers: {
                    'Content-Type': 'multipart/form-data'
                }
            };
            axios.post(self.host, formData,config).then(function(response) {  
                console.log('下标为'+index+"的文件已经上传完成");
                if(index == self.form.fileList.length-1){
                    self.afterUpload();
                    self.resetStatus();
                };
            }).catch(function(error) {  
                Message({
                    message: error,
                    type: 'error',
                    duration: 3 * 1000
                }); 
            })
        },
        resetStatus(){
            this.form.fileList = {};//选择的上传文件filelist对象
            this.form.storeList = [];//返回的数据（oldName,newName,url,size）
            this.form.name = '';
            this.host = '';//上传路径
            this.expire = 0;//上传秘钥有效时间戳
            this.key = '';//上传路径 + 文件名（后缀）
            this.isUpload = false;
            this.uploadOpts.policy = '';
            this.uploadOpts.OSSAccessKeyId = '';
            this.uploadOpts.success_action_status = 200;
            this.uploadOpts.signature = '';
        },
        /* 
            以下为回调方法（通知给父组件的方法） 
        */
        //上传前 抛出方法告诉父组件
        beforeUpload(filelist){
            this.$emit("before-upload",filelist);
        },
        //文件预览 返回相应的数据
        onPreview(list){
            this.$emit("on-preview",list);
        },
        // 所有文件上传完成 
        afterUpload(){
            this.$emit("after-upload",this.form.storeList);
        }

    }
}
</script>

<style lang="scss" scoped>
.uploader-box{
    display:inline-block;
    cursor:pointer;
}
input.el-input__inner{
    cursor:pointer !important;
}
.imgClass{
    max-width: 60px;
    max-height: 60px;
}
</style>