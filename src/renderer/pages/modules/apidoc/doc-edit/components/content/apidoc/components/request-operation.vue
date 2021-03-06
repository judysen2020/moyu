/*
    创建者：shuxiaokai
    创建时间：2020-10-13 13:47
    模块名称：请求发送相关
    备注：xxxx
*/
<template>
    <div class="w-100">
        <!-- 请求操作区域 -->
        <div class="d-flex w-100">
            <s-v-input 
                    v-model="request.url.path"
                    placeholder="只需要输入接口地址，前面不需要加域名，加了会被忽略"
                    :error="urlError"
                    size="small"
                    @blur="formatUrl"
                    @keyup.enter.native.stop="formatUrl"
            >
                <div slot="prepend" class="request-input">
                    <el-select v-model="request.methods" value-key="name" @change="handleChangeRequestMethods">
                        <el-option v-for="(item, index) in enabledRequestMethods" :key="index" :value="item" :label="item.name"></el-option>
                    </el-select>
                </div>                        
            </s-v-input>
            <el-button v-if="!loading" type="success" size="small" @click="sendRequest">发送请求</el-button>
            <el-button v-if="loading" type="danger" size="small" @click="stopRequest">取消请求</el-button>
            <el-button :loading="loading2" type="primary" size="small" @click="saveRequest">保存接口</el-button>
            <el-button :loading="loading3" type="primary" size="small" class="mr-1" @click="publishRequest">发布接口</el-button>
            <el-dropdown trigger="click" class="mr-1">
                <el-button type="primary" size="small">
                    其他操作<i class="el-icon-arrow-down el-icon--right"></i>
                </el-button>
                <el-dropdown-menu slot="dropdown">
                    <el-dropdown-item @click.native="dialogVisible = true">全局变量</el-dropdown-item>
                    <el-dropdown-item @click.native="dialogVisible = true">内置参数</el-dropdown-item>
                    <el-dropdown-item @click.native="$emit('fresh')">接口刷新</el-dropdown-item>
                    <el-dropdown-item @click.native="handleOpenConfigPage">全局配置</el-dropdown-item>
                </el-dropdown-menu>
            </el-dropdown>
        </div>
        <!-- 请求参数展示 -->
        <pre class="w-100">{{ fullUrl }}</pre>
        <!-- 请求传参方式选择 -->
        <div class="w-100 mt-2">
            <el-radio-group v-model="request.requestType" @change="handleChangeRequestMIMEType">
                <el-radio 
                        v-for="(item, index) in enabledContentType"
                        :key="index"
                        :label="item.value"
                        :disabled="!currentReqeustLimit.enabledContenType.find(val => val === item.value)"
                >
                    {{ item.name }}
                </el-radio>
            </el-radio-group>
        </div>
        <s-variable-dialog v-if="dialogVisible" :visible.sync="dialogVisible" @change="handleVariableChange"></s-variable-dialog>
        <s-preset-params-dialog :visible.sync="dialogVisible2" @success="getPresetEnum"></s-preset-params-dialog>
    </div>         
</template>

<script>
import variableDialog from "../dialog/variable-manage"
import presetParamsDialog from "../dialog/preset-params"
import qs from "qs"
import { dfsForest, findParentNode } from "@/lib/index"
import deepmerge from "deepmerge"
import FormData from "form-data/lib/form_data"
import FileType from "file-type/browser"
import buffer from "buffer"
const Buffer = buffer.Buffer;
export default {
    name: "REQUEST_OPERATION",
    components: {
        "s-variable-dialog": variableDialog,
        "s-preset-params-dialog": presetParamsDialog,
    },
    props: {
        request: { //完整请求参数
            type: Object,
            default() {
                return {};
            }
        },
        dataReady: { //数据是否请求回来
            type: Boolean,
            default: false
        },
    },
    computed: {
        docRules() { //文档规则
            return this.$store.state.apidocRules;
        },
        variables() { //全局变量
            return this.$store.state.apidoc.variables || [];
        },
        remoteResponse() {  //远端返回数据结果
            return this.$store.state.apidoc.responseData;
        },
        currentSelectDoc() { //当前选中的doc
            return this.$store.state.apidoc.activeDoc[this.$route.query.id];
        },
        tabs() { //全部tabs
            return this.$store.state.apidoc.tabs[this.$route.query.id];
        },
        couldPublish() {
            return this.$store.state.apidoc.remoteResponseEqualToLocalResponse
        },
        enabledRequestMethods() {
            return this.$store.state.apidocRules.requestMethods.filter(val => val.enabled);
        },
        enabledContentType() {
            return this.$store.state.apidocRules.contentType.filter(val => val.enabled);
        },
        fullUrl() {
            if (this.request.requestType === "params") {
                let queryStr = "";
                this.request.requestParams.map((val) => {
                    if (val.key && val._select) {
                        queryStr = `${queryStr}&${val.key}=${val.value}`
                    }
                })
                queryStr = queryStr.replace(/&/, "")
                queryStr = `${ queryStr ? "?" : ""}${queryStr}`
                return this.request.url.host + this.request.url.path + queryStr;
            } else {
                return this.request.url.host + this.request.url.path;
            }
        }
    },
    data() {
        return {
            urlError: { //-----------------请求url错误
                error: false,
                message: "请求url不能为空"
            },
            currentReqeustLimit: { enabledContenType: [] }, //当前选中请求类型额外规则
            //=====================================其他参数====================================//
            loading: false, //-------------发送请求
            loading2: false, //------------保存接口loading
            loading3: false, //------------发布接口loading
            dialogVisible: false, //-------全局变量
            dialogVisible2: false, //------内置参数
        };
    },
    mounted() {
        window.addEventListener("keydown", this.shortcutSave)
    },
    beforeDestroy() {
        window.removeEventListener("keydown", this.shortcutSave)
    },
    methods: {
        //=====================================获取数据====================================//
        //预设参数
        getPresetEnum() {

        },
        //获取联想参数枚举
        getMindParamsEnum() {
            this.$store.dispatch("apidoc/getMindParamsEnum", {
                projectId: this.$route.query.id,
            });
        },
        //===============================发送请求，保存请求，发布请求=======================//
        //发送请求
        sendRequest() {
            const foo = async (resolve, reject) => {
                const isValidateRequest = this.validateParams();
                if (isValidateRequest) {
                    this.loading = true;
                    const formData = new FormData();
                    const copyRequestInfo = deepmerge({}, this.request, { //数据拷贝,防止数据处理过程中改变拷贝请求参数的值
                        isMergeableObject(val) {
                            return typeof val === "object" && Object.prototype.toString.call(val).slice(8, -1) !== "Object"
                        }
                    });
                    const requestParams = this.convertPlainParamsToTreeData(copyRequestInfo.requestParams, true); //跳过未选中的参数
                    const headerParams = this.convertPlainParamsToTreeData(copyRequestInfo.header);
                    const url = copyRequestInfo.url.host + copyRequestInfo.url.path;
                    const method = copyRequestInfo.methods.toLowerCase();
                    const headers = headerParams;
                    delete headers["content-type"];
                    delete headers["Content-type"];
                    delete headers["content-Type"];
                    delete headers["Content-Type"];
                    let data = requestParams;
                    /*eslint-disable indent*/ 
                    switch (this.request.requestType) {
                        case "params":
                            headers["content-type"] = "application/json"
                            break;
                        case "json":
                            headers["content-type"] = "application/json"
                            break;
                        case "formData":
                            headers["content-type"] = formData.getHeaders()["content-type"];
                            for (const key in data) {
                                const hasOwn = Object.hasOwnProperty;
                                if (hasOwn.call(data, key)) {
                                    if (data[key].constructor.name === "ArrayBuffer") {
                                        const type = await FileType.fromBuffer(data[key]);
                                        formData.append(key, Buffer(data[key]), { contentType: type.mime });
                                    } else {
                                        formData.append(key, data[key]);
                                    }
                                }
                            }
                            data = formData;
                            break;
                        case "x-www-form-urlencoded":
                            headers["content-type"] = "application/x-www-form-urlencoded"
                            // for (const key in data) {
                            //     const hasOwn = Object.hasOwnProperty;
                            //     if (hasOwn.call(data, key)) {
                            //         if (data[key].constructor.name === "ArrayBuffer") {
                            //             const type = await FileType.fromBuffer(data[key]);
                            //             formData.append(key, Buffer(data[key]), { contentType: type.mime });
                            //         } else {
                            //             formData.append(key, data[key]);
                            //         }
                            //     }
                            // }
                            // data = formData;
                            break;
                        default:
                            headers["content-type"] = "application/json"
                            break;
                    }
                    let storeCookie = localStorage.getItem("pages/cookies") || "{}";
                    storeCookie = JSON.parse(storeCookie);
                    storeCookie = storeCookie[this.$route.query.id];
                    if (storeCookie) {
                        headers["cookie"] = storeCookie;
                    }
                    this.$store.dispatch("apidoc/sendRequest", { url, method, headers, data, requestType: this.request.requestType }).then(res => {
                        this.checkResponseParams(); //参数校验
                        this.injectCookie(res); //cookie注入
                        resolve();
                    }).catch(err => {
                        console.error(err);
                        reject(err)
                    }).finally(() => {
                        this.loading = false;
                    });
                } else {
                    reject("校验错误2")             
                }
            }
            return new Promise((resolve, reject) => {
                foo(resolve, reject);
            })
        },
        //取消发送
        stopRequest() {
            this.loading = false;
            this.$store.dispatch("apidoc/stopRequest");
            this.$store.commit("apidoc/changeLoading", false);
        },
        //保存接口
        saveRequest() {
            return new Promise((resolve, reject) => {
                const validParams = this.validateParams();
                if (validParams) {
                    const params = {
                        _id: this.currentSelectDoc._id,
                        projectId: this.$route.query.id,
                        item: {
                            requestType: this.request.requestType,
                            methods: this.request.methods,
                            url: {
                                host: this.request.url.host, 
                                path: this.request.url.path, 
                            },
                            requestParams: this.request.requestParams,
                            responseParams: this.request.responseParams,
                            header: this.request.header, 
                            description: this.request.description, 
                        }
                    };
                    this.saveMindParams(); //保存快捷联想参数
                    this.loading2 = true;
                    //取消未保存小圆点
                    this.$store.commit("apidoc/changeCurrentTabById", {
                        _id: this.currentSelectDoc._id,
                        projectId: this.$route.query.id,
                        changed: false
                    });
                    this.axios.post("/api/project/fill_doc", params).then(() => {
                        this.$store.commit("apidoc/changeTabInfoById", {
                            projectId: this.$route.query.id,
                            _id: this.currentSelectDoc._id,
                            method: this.request.methods,
                        })
                        this.$store.commit("apidoc/changeDocEditInfo", {
                            description: params.item.description,
                            header: params.item.header,
                            methods: params.item.methods,
                            requestParams: params.item.requestParams,
                            responseParams: params.item.responseParams,
                            url: params.item.url
                        }); //改变文档编辑内容，用于判断文档值是否发生了改变
                        this.getMindParamsEnum();
                        resolve();
                    }).catch(err => {
                        this.$errorThrow(err, this);
                        this.$store.commit("apidoc/changeCurrentTabById", {
                            _id: this.currentSelectDoc._id,
                            projectId: this.$route.query.id,
                            changed: true
                        });
                        reject();
                    }).finally(() => {
                        this.loading2 = false;
                    });                 
                } else {
                    reject();
                }                 
            })
        },
        //发布接口
        async publishRequest() {
            this.loading3 = true;
            try {
                await this.sendRequest();
                
                if (this.couldPublish) {
                    this.axios.put("/api/project/publish_doc", { _id: this.currentSelectDoc._id }).then((res) => {
                        this.$message.success("发布成功");
                        this.$store.commit("apidoc/changeDocInfo", res.data)
                    }).catch(err => {
                        this.$errorThrow(err, this);
                    }).finally(() => {
                        this.loading3 = false;
                    });
                } else {
                    this.$message.error("校验不通过无法发布接口");
                    this.loading3 = false;
                }                   
            } catch (error) {
                console.error(error);
                this.$message.error("校验错误")
                this.loading3 = false;
            }
        },
        //快捷保存
        shortcutSave(e) {
            if (this.tabs && this.tabs.length > 0 && e.ctrlKey && e.key === "s" && this.loading2 === false) {
                e.preventDefault();
                e.stopPropagation();
                this.saveRequest()
            }
        },
        //=====================================联想参数====================================//
        //保存联想参数(将之前录入过的参数保存起来)
        saveMindParams() {
            const mindRequestParams = [];
            const mindResponseParams = [];
            dfsForest(this.request.responseParams, {
                rCondition(value) {
                    return value.children;
                },
                rKey: "children",
                hooks: (data) => {
                    if (data.key !== "" && data.value !== "" && data.description !== "") {
                        const copyData = JSON.parse(JSON.stringify(data));
                        mindResponseParams.push(copyData);
                    }
                    if (data.key !== "" && (data.type === "object" || data.type === "array") && data.description !== "") {
                        const copyData = JSON.parse(JSON.stringify(data));
                        copyData.children = []; //只记录扁平数据
                        mindResponseParams.push(copyData);
                    }
                }
            });
            dfsForest(this.request.requestParams, {
                rCondition(value) {
                    return value.children;
                },
                rKey: "children",
                hooks: (data) => {
                    if (data.key !== "" && data.value !== "" && data.description !== "") {
                        mindRequestParams.push(data);
                    }
                }
            });
            const projectId = this.$route.query.id;
            const params = {
                projectId,
                mindRequestParams,
                mindResponseParams,
            };
            this.axios.post("/api/project/doc_params_mind", params).then(() => {
                
            }).catch(err => {
                this.$errorThrow(err, this);
            });
        },
        //=====================================数据处理和校验====================================//
        //将扁平数据转换为树形结构数据
        convertPlainParamsToTreeData(plainData, jumpChecked) {
            const result = {};
            const foo = (plainData, result) => {
                for(let i = 0,len = plainData.length; i < len; i++) {
                    if (jumpChecked && !plainData[i]._select) { //若请求参数未选中则不发送请求
                        continue;
                    }
                    const key = plainData[i].key.trim();
                    const value = this.convertVariable(plainData[i].value);
                    const type = plainData[i].type;
                    const valueTypeIsArray = Array.isArray(result);
                    const isComplex = (type === "object" || type === "array");
                    let arrTypeResultLength = 0; //数组类型值长度，用于数组里面嵌套对象时候对象取值
                    if (!isComplex && (key === "" || value === "")) { //非复杂数据需要填写参数名称才可以显示
                        continue
                    }
                    /*eslint-disable indent*/ 
                    switch (type) {
                        case "number": //数字类型需要转换为数字，转换前所有值都为字符串
                            valueTypeIsArray ? result.push(Number(value)) : result[key] = Number(value);
                            break;
                        case "boolean": //字符串类型不做处理
                            valueTypeIsArray ? result.push(result[key] = (value === "true" ? true : false)) : (result[key] = (value === "true" ? true : false));
                            break;
                        case "object":
                            valueTypeIsArray ? (arrTypeResultLength = result.push({})) : (result[key] = {});
                            if (plainData[i].children && plainData[i].children.length > 0) {
                                foo(plainData[i].children, valueTypeIsArray ? (result[arrTypeResultLength - 1]) : result[key]);
                            }
                            break;
                        case "array":
                            result[key] = [];
                            if (plainData[i].children && plainData[i].children.length > 0) {
                                foo(plainData[i].children, result[key]);
                            }
                            break;
                        default: //字符串或其他类型类型不做处理
                            valueTypeIsArray ? result.push(value) : (result[key] = value);
                            break;
                    }
                }
            }
            foo(plainData, result);
            return result;
        },
        //数据校验
        validateParams() {
            let isValidRequest = true;
            //=====================================检查请求url====================================//
            // if (this.request.url.path.trim() === "") { 
            //     this.urlError.error = true;
            //     this.urlError.message = "请求url不能为空";
            //     isValidRequest = false;
            // } else 
            if (!this.request.url.host) {
                this.$message.error("请选择请求服务器");
                isValidRequest = false;
            } else {
                this.urlError.error = false;
                isValidRequest = true;
            }
            //===========================检查参数是否必填或者按照规范填写======================//
            const validate =  (rawData) => {
                const deepMap = (requestData) => {
                    for (let i = 0, len = requestData.length; i < len; i++) {
                        const params = requestData[i];
                        if (params.children) {
                            deepMap(params.children);
                        }
                        if (i !== len - 1 || params.key.trim() !== "") { //最后一个数据并且未填写值则不做处理
                            const valueType = params.type;
                            const parentNode = findParentNode(params.id, rawData);
                            const isParentArray = (parentNode && parentNode.type === "array"); //父元素为数组，不校验key因为数组元素不必填写key值
                            const isComplex = (valueType === "object" || valueType === "array" || valueType === "file"); //自身类型为复杂类型不校验参数值，参数值写在树形组件下层
                            const key = params.key;
                            const value = this.convertVariable(params.value);
                            const description = params.description || "";
                            if (!isParentArray && (key.trim() === "" || key.match(/(^\s+)|(\s+$)/))) { //禁止包含空字符串(key)
                                this.$set(params, "_keyError", {
                                    error: true,
                                    message: "不能存在空白字符串"
                                });
                                isValidRequest = false;
                            }
                            if (!isComplex && !isParentArray) { //非对象，数组并且父元素不为数组，父元素为数组不必填写key和description
                                if (value.trim() === "" || value.match(/(^\s+)|(\s+$)/)) { //非空判断
                                    this.$set(params, "_valueError", {
                                        error: true,
                                        message: "不能存在空白字符串"
                                    });
                                    isValidRequest = false;
                                } else if (valueType === "number" && !this._isNumberLike(value)) {
                                    this.$set(params, "_valueError", {
                                        error: true,
                                        message: "参数值必须为数字类型"
                                    });
                                    isValidRequest = false;
                                }
                            }
                            if (!isComplex && !isParentArray && (description.trim() === "" || description.match(/(^\s+)|(\s+$)/))) { //禁止包含空字符串(description)
                                this.$set(params, "_descriptionError", {
                                    error: true,
                                    message: "不能存在空白字符串"
                                });
                                isValidRequest = false;
                            }
                        }
                    }
                }   
                deepMap(rawData);             
            }

            validate(this.request.requestParams);
            validate(this.request.header);

            if (!isValidRequest) {
                this.$nextTick(() => {
                    const errorIptDom = document.querySelector(".v-input.valid-error .el-input__inner");
                    errorIptDom ? errorIptDom.focus() : null;
                })
            }
            this.$store.commit("apidoc/changeParamsValid", isValidRequest)
            return isValidRequest;
        },
        //参数校验
        checkResponseParams() {
            // console.log(9, this.remoteResponse)
            if (this.remoteResponse.contentType && this.remoteResponse.contentType.includes("application/json")) {
                const remoteParams = JSON.parse(this.remoteResponse.value);
                const localParams = this.convertPlainParamsToTreeData(this.request.responseParams);
                let responseErrorType = null; //校验错误类型
                const hasOwn = Object.hasOwnProperty;

                if (Object.keys(localParams).length === 0) {
                    responseErrorType = "lackKey"
                }
                const foo = (localData, remoteData) => {
                    for (let i in localData) { //不处理原型链上的数据
                        if (!hasOwn.call(localData, i)) {
                            continue;
                        }
                        const remoteKeys = Object.keys(remoteData); //-----远程keys
                        const localKeys = Object.keys(localData); //-------本地keys
                        const isLackKey = localKeys.some(val => !remoteKeys.includes(val)); //远程结果是否缺少对应字段
                        const isTooMuchKey = !remoteKeys.every(val => localKeys.includes(val)); //远程结果是否超出字段
                        //字段超出或者缺少判断
                        if (isLackKey) {
                            responseErrorType = "lackKey";
                            return;
                        }   
                        if (isTooMuchKey) {
                            responseErrorType = "tooMuchKey"
                            return
                        }
                        //判断字段类型是否一致
                        const localValue = localData[i];
                        const remoteValue = remoteData[i];
                        const localType = this.getType(localValue);
                        const remoteType = this.getType(remoteValue);
                        if (localType !== remoteType) {
                            responseErrorType = "typeError"
                            return
                        }
                        if (localType === "object") {
                            foo(localValue, remoteValue);
                        }
                        if (localType === "array" && remoteValue[0]) {
                            foo(localValue[0], remoteValue[0]);
                        }
                    }                    
                }
                foo(localParams, remoteParams);
                if (responseErrorType) {
                    this.$store.commit("apidoc/changeCondition", false)
                } else {
                    this.$store.commit("apidoc/changeCondition", true)
                }
            }
        },
        //=====================================url操作====================================//
        //删除无效请求字符
        formatUrl() {
            this.convertQueryToParams();
            const protocolReg = /(\/?https?:\/\/)?/;
            // const dominReg = /[a-zA-Z0-9]+\./
            this.request.url.path = this.request.url.path.replace(protocolReg, ""); //去除掉协议
            this.request.url.path.startsWith(",") ? (this.request.url.path = "/" + this.request.url.path) : "";
            // const pathReg = /\/(?!\/)[^#\\?:.]+/; //查询路径正则
            // const matchedPath = this.request.url.path.match(pathReg);
            // if (matchedPath) {
            //     this.request.url.path = matchedPath[0];
            // } else if (this.request.url.path.trim() === "") {
            //     this.request.url.path = "/";
            // } else if (!this.request.url.path.startsWith("/")) {
            //     this.request.url.path = "/" + this.request.url.path;
            // }
            const queryReg = /\?.*/;
            this.request.url.path = this.request.url.path.replace(queryReg, "")
            //检查url是否为空
            // if (this.request.url.path.trim() === "") { 
            //     this.urlError.error = true;
            //     this.urlError.message = "请求url不能为空";
            // } else {
            //     this.urlError.error = false;
            // }
        },
        //将请求url后面查询参数转换为params
        convertQueryToParams() {
            let queryString = this.request.url.path.split("?") || "";
            queryString = queryString ? queryString[1] : "";
            const queryParams = qs.parse(queryString);
            for (const i in queryParams) {
                const reqParams = this.request.requestParams;
                if (!reqParams.find(val => val.key === i)) {
                    this.request.requestParams.unshift({
                        id: this.uuid(),
                        key: i, //--------------请求参数键
                        value: queryParams[i], //------------请求参数值
                        type: "string", //-------------请求参数值类型
                        description: "", //------描述
                        required: true, //-------是否必填
                        children: [], //---------子参数
                        _select: true, //默认选中
                    })
                }
            }
            const matchedComponent = this.getComponentByName("REQUEST_PARAMS");
            matchedComponent.selectChecked();
        },
        //改变请求方法
        handleChangeRequestMethods(val) {
            this.currentReqeustLimit = val;
            if (val.value === "get") { //get请求需要清空嵌套数据
                this.request.requestParams.forEach(params => {
                    params.children = [];
                    params.type = "string";
                })
                this.request.requestType = "params"; 
            } else {
                if (!val.enabledContenType.includes(this.request.requestType)) {
                    this.request.requestType = val.enabledContenType[0];
                }
            } 
            this.request.methods = val.value;
            //改变tabs导航请求方式
            this.$store.commit("apidoc/changeTabInfoById", {
                _id: this.currentSelectDoc._id,
                projectId: this.$route.query.id,
                method: val.value
            });
            //改变banner请求方式
            this.$store.commit("apidoc/changeDocBannerInfoById", {
                id: this.currentSelectDoc._id,
                method: val.value
            });
            this.handleChangeRequestMIMEType(this.request.requestType);
        },
        //改变MimeType
        handleChangeRequestMIMEType(val) {
            if (val === "formData") {
                this.request.header.forEach(header => {
                    if (header.key.toLowerCase() === "content-type") {
                        header.value = "multipart/form-data"
                    }
                })
            } else if (val === "json") {
                this.request.header.forEach(header => {
                    if (header.key.toLowerCase() === "content-type") {
                        header.value = "application/json; charset=utf-8"
                    }
                })
            } else if (val === "params") { //查询类型application无意义
                this.request.header.forEach(header => {
                    if (header.key.toLowerCase() === "content-type") {
                        header.value = "application/json; charset=utf-8"
                    }
                })
            } else if (val === "x-www-form-urlencoded") {
                this.request.header.forEach(header => {
                    if (header.key.toLowerCase() === "content-type") {
                        header.value = "application/x-www-form-urlencoded"
                    }
                })
            }
        },
        //hack通过改变_variableChange触发watch事件刷新变量值
        handleVariableChange() {
            this.request._variableChange = !this.request._variableChange;
        },
        //=====================================其他操作====================================//
        //修正contentType
        async fixContentType() {
            if (this.docRules.cacheProjectId !== this.$route.query.id) {
                await this.$store.dispatch("apidocRules/getRuels", {
                    projectId: this.$route.query.id
                });
            }
            this.currentReqeustLimit = this.docRules.requestMethods.find(val => val.value === this.request.methods);
            if (!this.currentReqeustLimit.enabledContenType.includes(this.request.requestType)) { //修正不合法的请求类型，默认取合法请求类型的第一个
                this.request.requestType = this.currentReqeustLimit.enabledContenType[0];
            }
        },
        //cookie注入
        injectCookie(res) {
            const cookies = res.headers["set-cookie"] || [];
            const cookie = cookies.join(",");
            if (cookie) {
                let storeCookie = localStorage.getItem("pages/cookies") || "{}";
                storeCookie = JSON.parse(storeCookie);
                storeCookie[this.$route.query.id] = cookie;
                localStorage.setItem("pages/cookies", JSON.stringify(storeCookie))
            }
        },
        //改变变量信息
        convertVariable(val) {
            if (val == null) {
                return;
            }
            const matchedData = val.toString().match(/{{\s*(\w+)\s*}}/);
            if (val && matchedData) {
                const varInfo = this.variables.find(v => {
                    return v.name === matchedData[1];
                });
                if (varInfo) {
                    return val.replace(/{{\s*(\w+)\s*}}/, varInfo.value);
                } else {
                    return val;
                }
            } else {
                return val;
            }
        },
        //获取参数类型
        getType(value) {
            if (typeof value === "string") {
                return "string"
            } else if (typeof value === "number") { //NaN
                return "number"
            } else if (typeof value === "boolean") {
                return "boolean"
            } else if (Array.isArray(value)) {
                return "array"
            } else if (typeof value === "object" && value !== null) {
                return "object"
            } else { // null undefined ...
                return "string"
            }
        },
        //打开配置界面
        handleOpenConfigPage() {
            const configTabInfo = {
                _id: "idConfig",
                projectId: this.$route.query.id,
                docName: "文档全局配置",
                tabType: "config"                
            }
            this.$store.commit("apidoc/addTab", {
                projectId: this.$route.query.id,
                ...configTabInfo
            });
            this.$store.commit("apidoc/changeCurrentTab", {
                projectId: this.$route.query.id,
                activeNode: configTabInfo
            });
        }
    }
};
</script>



<style lang="scss">
.request-input {
    display: flex;
    align-items: center;
    .el-select {
        width: 100px;
    }
}
</style>
