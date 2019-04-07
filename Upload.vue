/*jshint -W065 */
<template lang="html">
<div class="upload">
    <form style="display: none" name=theform>
        <a type="radio" name="myradio" ></a> 上传文件名字保持本地文件名字
        <a type="radio" name="myradio" ></a> 上传文件名字是随机文件名, 后缀保留
    </form>
    <h4>您所选择的文件列表：</h4>
    <div id="ossfile"></div>
    <br/>
    <div id="container" v-show="disabled">
        <a id="selectfiles" href="javascript:void(0);" class='btn' >选择文件</a>
    </div>
    <div v-show="!disabled">
        <a  href="javascript:void(0);" class='btn' style="color:#fff;background-color: #ccc;border-color: #ccc;" disabled >选择文件</a>
    </div>
    <pre id="console">{{message}}</pre>
    <p>&nbsp;</p>
</div>
</template>

<script>
import MD5 from '../common/MD5.js'
import plupload from '../../../static/js/plupload.full.min.js';
import {
    mapGetters,
    mapActions
} from 'vuex'
export default {
    name: 'upload',
    data() {
        return {
            ossFile: '',
            uploadMethod: '',
            accessid: '',
            accesskey: '',
            host: '',
            policyBase64: '',
            signature: '',
            callbackbody: '',
            filename: '',
            key: '',
            expire: 0,
            g_object_name: '',
            g_object_name_type: '',
            now: Date.parse(new Date()) / 1000,
            message: '',
            signatureNonce: null,
            aodianAccessId: '',
            expires: null,
            signature: '',
            filelength: 0,
            disabled: true,
            setTimeout:null,
            disabled_length:0,
            btns:null
        };
    },
    props: {
        fileId: '',
        beforeUpload: Function,
        onSuccess: Function,
        onError: Function,
        onProgress: Function,
    },
    components: {
        plupload,
        MD5
    },
    watch:{
        disabled_length(newval){
            if(newval==0){
                this.disabled=true
            }
        }
    },
    computed: {
        ...mapGetters({
            'userInfo': 'info/userInfo',
        })
    },
    beforeCreate() {},
    mounted() {
        this.$nextTick(() => {
            this.upload();
        });
    },
    methods: {
        startup() {
            this.$Notice.success({
                title: '上传已开始,请不要离开当前页',
                duration: 2
            });
        },
        sendRequest() {
            const xmlhttp = new XMLHttpRequest();
            // 你的服务端接口地址:  参考demo:http://oss-demo.aliyuncs.com/oss-h5-upload-js-php/
            // 服务端签名后直传文档:  https://help.aliyun.com/document_detail/31926.html
            const serverUrl = 'http://oss-demo.aliyuncs.com/oss-h5-upload-js-php/php/get.php';
            xmlhttp.open('GET', serverUrl, false);
            xmlhttp.send(null);
            return xmlhttp.responseText;
        },
        getSignature() {
            // 可以判断当前expire是否超过了当前时间,如果超过了当前时间,就重新取一下.3s 做为缓冲
            this.now = Date.parse(new Date()) / 1000;
            if (this.expire < this.now + 3) {
                const body = this.sendRequest();
                const e = eval;
                const obj = e(`(${body})`);
                this.host = obj.host;
                this.policyBase64 = obj.policy;
                this.accessid = obj.accessid;
                this.signature = obj.signature;
                this.expire = parseInt(obj.expire, 10);
                this.callbackbody = obj.callback;
                this.key = obj.dir;
                return true;
            }
            return false;
        },
        checkObjectRadio() {
            const tt = document.getElementsByName('myradio');
            for (let i = 0; i < tt.length; i += 1) {
                if (tt[i].checked) {
                    this.g_object_name_type = tt[i].value;
                    break;
                }
            }
        },
        randomString(len = 32) {
            const chars = 'ABCDEFGHJKMNPQRSTWXYZabcdefhijkmnprstwxyz2345678';
            const maxPos = chars.length;
            let pwd = '';
            for (let i = 0; i < len; i += 1) {
                pwd += chars.charAt(Math.floor(Math.random() * maxPos));
            }
            return pwd;
        },
        getSuffix(filename) {
            const pos = filename.lastIndexOf('.');
            let suffix = '';
            if (pos !== -1) {
                suffix = filename.substring(pos);
            }
            return suffix;
        },
        calculateObjectName(filename) {
            if (this.g_object_name_type === 'local_name') {
                this.g_object_name += `${filename}`;
            } else if (this.g_object_name_type === 'random_name') {
                const suffix = this.getSuffix(filename);
                this.g_object_name = this.key + this.randomString(10) + suffix;
            }
            return '';
        },
        getUploadedObjectName(filename) {
            if (this.g_object_name_type === 'local_name') {
                let tmpName = this.g_object_name;
                tmpName = tmpName.replace(`${filename}`, filename);
                return tmpName;
            } else if (this.g_object_name_type === 'random_name') {
                return this.g_object_name;
            }
            return '';
        },
        setUploadParam(up, ret) {
            let datetime = new Date().getTime()
            if (!ret) {
                up.files.map((v, i) => {
                    var index = v.name.indexOf('.')
                    v.name = v.name.slice(0, index) + '_upload_' + datetime + i + v.name.slice(index)
                    // console.log(v.name)
                })

            }

            const newMultipartParams = {
                access_id: this.aodianAccessId,
                expires: this.expires,
                signature: this.signature,
                signature_nonce: this.signatureNonce,
            };
            up.setOption({
                multipart_params: newMultipartParams,
            });
            up.start();
        },
        upload() {
            const that = this;
            let hostname = document.location.protocol
            
            const uploader = new plupload.Uploader({
                runtimes: 'html5,flash,silverlight,html4',
                browse_button: 'selectfiles',
                multi_selection: true,
                container: document.getElementById('container'),
                flash_swf_url: '../../static/pupload/plupload-2.1.2/js/Moxie.swf',
                silverlight_xap_url: '../../static/pupload/plupload-2.1.2/js/Moxie.xap',
                // url:`//upload-dvr.aodianyun.com/v2/DVR.FormUpload`,
                url:`${hostname}//upload-dvr.aodianyun.com/v2/DVR.FormUpload`,
                chunk_size: '5mb',
                max_retries: 3,
                filters: {
                    mime_types: [{
                        title: '允许上传文件类型',
                        extensions: 'avi,f4v,mpg,mp4,flv,wmv,mov,3gp,rmvb,mkv,asf,ts,mts,dat,vob,mp3,wav,m4v,webm',
                    }],
                    max_file_size: '99999mb',
                    // 不允许队列中存在重复文件1
                    prevent_duplicates: true,
                },
                init: {
                    PostInit: () => {
                       
                    },
                    FilesAdded: (up, files) => {
                         console.log('FilesAdded')
                        let date = new Date() / 1000
                        var datetime = this.$root.dayr(date)
                        this.filelength = up.files.length
                        plupload.each(files, (file) => {
                                 let index = file.name.indexOf('.')
                                let val = file.name.slice(index)
                                let datetimeLength = new Date().getTime().toString().length + 30 + 8 + val.length
                                if (file.name.length > datetimeLength) {
                                        this.message = file.name.slice(0,30)+'...' +' 文件名太长，请不要超过30个字!';
                                        document.getElementById('ossfile').innerHTML += `
                                            <div id="${file.id}" class="ossdiv">
                                            <span title="${file.name}(${plupload.formatSize(file.size)})">${file.name+'  '+datetime} (${plupload.formatSize(file.size)})</span>
                                            <b style="color:red">文件名超出限制</b>
                                            <div class="progress"><div class="progress-bar" style="width: 0"></div>
                                            </div>
                                            <button class="deleBtn" style="padding: 4px 6px;border-radius: 4px;">删除</button>
                                            </div>`;
                                         this.disabled=false
                                    } else{
                                        document.getElementById('ossfile').innerHTML += `
                                            <div id="${file.id}" class="ossdiv">
                                            <span title="${file.name}(${plupload.formatSize(file.size)})">${file.name+'  '+datetime} (${plupload.formatSize(file.size)})</span>
                                            <b>准备上传</b>
                                            <div class="progress"><div class="progress-bar" style="width: 0"></div>
                                            </div>
                                            <button class="nodeleBtn" style="padding: 4px 6px;border-radius: 4px;">删除</button>
                                            </div>`;
                                    }  
                        });
                        if(this.disabled){
                            this.setUploadParam(up, false)
                        }
                        let seft =this
                        this.btns =  document.getElementsByClassName('deleBtn')
                         let ind = 0
                        for(let i = 0;i<this.btns.length;i++){
                             this.btns[i].onclick = function(e){
                                 let k = i-ind>0?i-ind:0
                                   var a = seft.btns[k].parentNode;
                                    var divs = document.getElementById('ossfile').getElementsByClassName('ossdiv');
                                    for(var index = 0, l = divs.length;k < l; index++ ){
                                        if(a == divs[index]){
                                             up.removeFile(up.files[index]);
                                             seft.btns[k].parentNode.remove()
                                             seft.message=''
                                             ind++
                                        break;
                                    }
                                    }
                                }
                        }
                    },
                    FilesRemoved:(up, file)=>{
                        this.$nextTick(()=>{
                             this.disabled_length = document.getElementsByClassName('deleBtn').length
                             if(this.disabled_length==0){
                                 this.disabled=true
                                 this.setUploadParam(up, false)
                            }
                        })
                    },
                    BeforeUpload: (up, file) => {
                    
                                   
                           
                    },
                    UploadProgress: (up, file) => {
                        this.message = '文件上传中...，请勿离开当前页面';
                        const d = document.getElementById(file.id);
                        d.getElementsByTagName('b')[0].innerHTML = `<span>${file.percent}%</span>`;
                        const prog = d.getElementsByTagName('div')[0];
                        const progBar = prog.getElementsByTagName('div')[0];
                        progBar.style.width = `${2 * file.percent}px`;
                        progBar.setAttribute('aria-valuenow', file.percent);
                    },
                    FileUploaded: (up, file, info) => {
                        this.message = '';
                        if (info.status === 200) {
                            document.getElementById(file.id).getElementsByTagName('b')[0].innerHTML = `上传成功`;
                            this.$Notice.success({
                                title: file.name + '上传成功 ',
                                duration: 1
                            });
                            setTimeout(()=>{
                                this.$root.Bus.$emit('FileUpLoading')
                            },1000)
                        } else {
                            document.getElementById(file.id).getElementsByTagName('b')[0].innerHTML = info.response.FlagString;
                        }
                    },
                    
                    Error: (up, err) => {
                        this.$Notice.error({
                            title: '上传失败 ',
                            duration: 1
                        });
                        if (err.code === -600) {
                            that.message = '文件大小超出限制，单文件不能超过4G';
                        } else if (err.code === -602) {
                            that.message = '文件重复上传，请刷新页面后重新上传文件!';
                        }else if (err.code === -200) {
                            that.message = 'http网络错误，请刷新页面后重新上传文件!';
                        }else if (err.code === -300) {
                            that.message = '磁盘读写错误，请刷新页面后重新上传文件!';
                        }else if (err.code === -400) {
                            that.message = '存在安全问题!';
                        }else if (err.code === -500) {
                            that.message = '初始化失败，请刷新页面后重新上传文件!';
                        }else if (err.code === -702) {
                            that.message = '文件大小超过了plupload所能处理的最大值!';
                        }else if (err.code === 403) {
                            that.message = '页面失效，请刷新页面后重新上传文件!';
                        } else {
                            that.message = err.message;
                        }
                    },
                },
            });
            uploader.init();
        },
        getuploadInfo() {
            this.$Api.get('uploadAccessInfo').then(res => {
                if (res.code == 200 && res.errorCode == 0) {
                    this.aodianAccessId = res.data.aodianAccessId
                    this.expires = res.data.expires
                    this.signature = res.data.signature
                    this.signatureNonce = res.data.signatureNonce
                } else {
                    this.$Message.error(res.errorMessage)
                }
            })
        }
    },
    created() {
        this.getuploadInfo()
    }
};
</script>

<style src="./upload.css"></style><style scoped>
.btn.active {
    background: #F2f4f6;
    border-color: #f2f4f6;
}

</style>
