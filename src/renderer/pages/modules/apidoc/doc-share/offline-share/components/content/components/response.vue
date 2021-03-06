/*
    创建者：shuxiaokai
    创建时间：2020-07-10 20:13
    模块名称：请求返回信息
    备注：xxxx
*/
<template>
    <div class="response">
        <!-- 基本信息 -->
        <s-collapse title="基本信息" class="baseInfo">
            <div class="py-2">
                <s-label-value label="请求地址：" title="鼠标左键拷贝路径，鼠标右键拷贝全部" class="d-flex">
                    <span 
                        v-copy="requestData.url.path" 
                        v-copy2="requestData.url.host + requestData.url.path"
                        class="text-ellipsis">
                        {{ requestData.url.host + requestData.url.path }}
                    </span>
                </s-label-value>
                <s-label-value label="请求方式：" class="d-flex">
                    <span class="green">{{ requestData.methods.toUpperCase() }}</span>
                </s-label-value>
                <div class="d-flex mt-2">
                    <div class="d-flex a-center">
                        <span>状态码:&nbsp;</span>
                        <span 
                            v-if="remoteResponse.status"
                            :class="{
                                green: remoteResponse.status >= 200 && remoteResponse.status < 300,
                                orange: remoteResponse.status >= 300 && remoteResponse.status < 400,
                                red: remoteResponse.status >= 400
                            }"
                        >
                            {{ remoteResponse.status }}
                        </span>
                        <span v-else title="未请求数据" class="el-icon-question gray-500"></span>
                    </div>
                    <div class="mx-2 gray-500">|</div>
                    <div class="d-flex a-center">
                        <span>时长:&nbsp;</span>
                        <span 
                            v-if="remoteResponse.rt"
                            :class="{
                                green: remoteResponse.rt >= 0 && remoteResponse.rt < 2000,
                                orange: remoteResponse.rt >= 2000 && remoteResponse.rt < 5000,
                                red: remoteResponse.rt >= 5000
                            }"
                        >
                            {{ (remoteResponse.rt / 1000).toFixed(2) }}s
                        </span>
                        <span v-else title="未请求数据" class="el-icon-question gray-500"></span>
                    </div>
                    <div class="mx-2 gray-500">|</div>
                    <div class="d-flex a-center">
                        <span>大小:&nbsp;</span>
                        <span 
                            v-if="remoteResponse.size"
                            :class="{
                                green: remoteResponse.size >= 0 && remoteResponse.size < 10000,
                                orange: remoteResponse.size >= 10000 && remoteResponse.size < 15000,
                                red: remoteResponse.size >= 15000
                            }"
                        >
                            {{ size }}
                        </span>
                        <span v-else title="未请求数据" class="el-icon-question gray-500"></span>
                    </div>
                    <div class="mx-2 gray-500">|</div>
                    <div class="d-flex a-center">
                        <span>返回格式:&nbsp;</span>
                        <el-popover v-if="remoteResponse.contentType" placement="top-start" width="200" trigger="hover" :content="remoteResponse.contentType">
                            <span v-if="remoteResponse.contentType.includes('application/json')" slot="reference" class="theme-color">JSON</span>
                            <span v-else-if="remoteResponse.contentType.includes('image/')" slot="reference" class="theme-color">图片({{ remoteResponse.contentType.replace(/image\//, "") }})</span>
                            <span v-else-if="remoteResponse.contentType.includes('application/pdf')" slot="reference" class="theme-color">pdf</span>
                            <span v-else-if="remoteResponse.contentType.includes('application/vnd.ms-excel')" slot="reference" class="theme-color">Excel</span>
                            <span v-else-if="remoteResponse.contentType.includes('application/vnd.openxmlformats-officedocument.spreadsheetml.sheet')" slot="reference" class="theme-color">Excel</span>
                            <span v-else-if="remoteResponse.contentType.includes('text')" slot="reference" class="theme-color">文本({{ remoteResponse.contentType.replace(/text\//, "") }})</span>
                            <span v-else-if="remoteResponse.contentType.includes('application/vnd.openxmlformats-officedocument.wordprocessingml.document')" slot="reference" class="theme-color">Word</span>
                            <span v-else-if="remoteResponse.contentType.includes('application/msword')" slot="reference" class="theme-color">Word</span>
                            <span v-else slot="reference" class="theme-color">{{ remoteResponse.contentType }}</span>
                        </el-popover>
                        <span v-else title="未请求数据" class="el-icon-question gray-500" slot="reference"></span>
                    </div>
                </div>
            </div>
        </s-collapse>
        <!-- 请求头 -->
        <s-collapse title="请求头">
            <template v-if="requestData.header.length > 1">
                <template v-for="(item, index) in requestData.header">
                    <s-label-value v-if="item.key" :label="item.key + '：'" :value="convertVariable(item.value)" :key="index" class="w-100" label-width="auto"></s-label-value>
                </template>
            </template>
            <div v-else class="f-xs gray-500">暂无数据</div>
        </s-collapse>
        <!-- 请求参数 -->
        <s-collapse title="请求参数" :active="false">
            <s-tree-json :data="selectedRequestParams"></s-tree-json>
        </s-collapse>
        <!-- 响应参数 -->
        <s-collapse title="响应参数" :active="false">
            <s-tree-json :data="requestData.responseParams"></s-tree-json>
        </s-collapse>
        <!-- 远程结果 -->
        <s-collapse title="远程结果">
            <div v-loading="loading" class="response-wrapper">
                <div v-if="remoteResponse && remoteResponse.contentType">
                    <!-- json -->
                    <s-json v-if="remoteResponse.contentType.includes('application/json')" :data="remoteResponse.data" :check-data="responseParams" @export="handleExport"></s-json>
                    <!-- svg -->
                    <span v-else-if="remoteResponse.contentType.includes('image/svg+xml')" v-html="remoteResponse.data"></span>
                    <!-- 图片 -->
                    <el-image 
                        v-else-if="remoteResponse.contentType.includes('image/')"
                        class="img-style"
                        :src="remoteResponse.data.blobUrl"
                        :preview-src-list="[remoteResponse.data.blobUrl]"
                        fit="scale-down"
                    >
                    </el-image>
                    <!-- 纯文本 -->
                    <pre v-else-if="remoteResponse.contentType.includes('text/')" v-text="remoteResponse.data" class="res-text"></pre>
                    <!-- pdf -->
                    <iframe v-else-if="remoteResponse.contentType.includes('application/pdf')" :src="remoteResponse.data.blobUrl" class="pdf-style"></iframe>
                    <!-- xls(低版本excel) -->
                    <div v-else-if="remoteResponse.contentType.includes('application/vnd.ms-excel')" class="office-style">
                        <svg class="svg-icon" aria-hidden="true" title="xls">
                            <use xlink:href="#iconexcel"></use>
                        </svg> 
                        <div>
                            <div>xls</div>
                            <s-download-button :url="remoteResponse.data.blobUrl" static>下载</s-download-button>
                        </div>
                    </div>
                    <!-- 高版本excel -->
                    <div v-else-if="remoteResponse.contentType.includes('application/vnd.openxmlformats-officedocument.spreadsheetml.sheet')" class="office-style">
                        <svg class="svg-icon" aria-hidden="true" title="xls">
                            <use xlink:href="#iconexcel"></use>
                        </svg> 
                        <div>
                            <div>xlsx</div>
                            <s-download-button :url="remoteResponse.data.blobUrl" static>下载</s-download-button>
                        </div>
                    </div>
                    <!-- docx -->
                    <div v-else-if="remoteResponse.contentType.includes('application/vnd.openxmlformats-officedocument.wordprocessingml.document')" class="office-style">
                        <svg class="svg-icon" aria-hidden="true" title="xls">
                            <use xlink:href="#iconWORD"></use>
                        </svg> 
                        <div>
                            <div>docx</div>
                            <s-download-button :url="remoteResponse.data.blobUrl" static>下载</s-download-button>
                        </div>
                    </div>
                    <!-- doc(低版本word) -->
                    <div v-else-if="remoteResponse.contentType.includes('application/msword')" class="office-style">
                        <svg class="svg-icon" aria-hidden="true" title="xls">
                            <use xlink:href="#iconWORD"></use>
                        </svg> 
                        <div>
                            <div>doc</div>
                            <s-download-button :url="remoteResponse.data.blobUrl" static>下载</s-download-button>
                        </div>
                    </div>
                    <div v-else class="d-flex f-lg">
                        <svg class="svg-icon" aria-hidden="true" title="xls">
                            <use xlink:href="#iconicon_weizhiwenjian"></use>
                        </svg> 
                        <div>
                            <div>{{ remoteResponse.contentType }}</div>
                            <s-download-button :url="remoteResponse.data.blobUrl" static>下载</s-download-button>
                        </div>
                    </div>
                </div>
            </div>
        </s-collapse>
        <s-collapse title="返回头">
            <pre v-if="remoteResponse">{{ remoteResponse.headers }}</pre>
        </s-collapse>
    </div>
</template>

<script>
import { dfsForest } from "@/lib/index"
import { formatBytes } from "@/lib"
export default {
    props: {
        requestData: {
            type: Object,
            default() {
                return {};
            }
        },
    },
    computed: {
        variables() { //全局变量
            return this.$store.state.apidoc.variables || [];
        },
        responseParams() { //返回参数(对象类型)
            const copyData = JSON.parse(JSON.stringify(this.requestData.responseParams)); //扁平数据拷贝
            const result = this.convertPlainParamsToTreeData(copyData);
            return result;
        },
        selectedRequestParams() { //只显示选中的json数据
            const copyData = JSON.parse(JSON.stringify(this.requestData.requestParams)); //扁平数据拷贝
            const result = [];
            const foo = (copyData, result) => {
                for(let i = 0; i < copyData.length; i++) {
                    const element = copyData[i];
                    const selectData = copyData[i]._select;
                    if (selectData) {
                        const index = result.push({
                            ...element,
                            children: []
                        })
                        if (element.children && element.children.length > 0) {
                            foo(element.children, result[index-1].children);
                        }
                    }
                }
            }
            foo(copyData, result);
            return result;
            // const copyData = JSON.parse(JSON.stringify(this.requestData.requestParams)); //扁平数据拷贝
            // const result = {};
            // dfsForest(copyData, {
            //     rCondition(value) {
            //         return value && value.children && value.children.length > 0;
            //     },
            //     rKey: "children",
            //     hooks: (val, i, forestData, parent) => {
            //         result

            //         if (val != null && !val._select) {
            //             if (!parent) {
            //                 console.log(val, i, copyData, 222)
            //                 copyData.splice(i, 1);
            //             } else {
            //                 parent.children.splice(i, 1);
            //             }
            //         }
            //     }
            // });
            // return copyData;
        },
        remoteResponse() {  //远端返回数据结果
            return this.$store.state.apidoc.responseData;
        },
        loading() { //是否正在请求数据
            return this.$store.state.apidoc.loading
        },
        size() { //当前返回值大小
            return formatBytes(this.$store.state.apidoc.responseData.size)
        },
        speed() { //实时请求速度
            return formatBytes(this.$store.state.apidoc.responseData.speed)
        },
    },
    data() {
        return {
            validError: false, //校验状态
        };
    },
    created() {
        
    },
    methods: {
        //=====================================组件间交互====================================//  
        //将扁平数据转换为树形结构数据
        convertPlainParamsToTreeData(plainData, jumpChecked) {
            const result = {};
            const foo = (plainData, result) => {
                for(let i = 0,len = plainData.length; i < len; i++) {
                    if (jumpChecked && !plainData[i]._select) { //若请求参数未选中则忽略掉
                        continue;
                    }
                    const key = plainData[i].key.trim();
                    const value = this.convertVariable(plainData[i].value);
                    const type = plainData[i].type;
                    const resultIsArray = Array.isArray(result);
                    const isComplex = (type === "object" || type === "array");
                    let arrTypeResultLength = 0; //数组类型值长度，用于数组里面嵌套对象时候对象取值
                    if (!isComplex && (key === "" || value === "")) { //非复杂数据需要填写参数名称才可以显示
                        continue
                    }
                    /*eslint-disable indent*/ 
                    switch (type) {
                        case "number": //数字类型需要转换为数字，转换前所有值都为字符串
                            resultIsArray ? result.push(Number(value)) : result[key] = Number(value);
                            break;
                        case "boolean": //字符串类型不做处理
                            resultIsArray ? result.push(result[key] = (value === "true" ? true : false)) : (result[key] = (value === "true" ? true : false));
                            break;
                        case "object":
                            resultIsArray ? (arrTypeResultLength = result.push({})) : (result[key] = {});
                            if (plainData[i].children && plainData[i].children.length > 0) {
                                foo(plainData[i].children, resultIsArray ? (result[arrTypeResultLength - 1]) : result[key]);
                            }
                            break;
                        case "array":
                            result[key] = [];
                            if (plainData[i].children && plainData[i].children.length > 0) {
                                foo(plainData[i].children, result[key]);
                            }
                            break;
                        default: //字符串或其他类型类型不做处理
                            resultIsArray ? result.push(value) : (result[key] = value);
                            break;
                    }
                }
            }
            foo(plainData, result);
            return result;
        },
        //导出数据
        handleExport(data) {
            const copyData =JSON.parse(JSON.stringify(data))
            dfsForest(copyData, {
                rCondition(value) {
                    return value.children;
                },
                rKey: "children",
                hooks: (val) => {
                    val.description || (this.$set(val, "description", ""))
                    Object.assign(val, {
                        id: this.uuid(),
                        required: true, //-------是否必填
                    })
                }
            });
            copyData.push(this.generateParams());
            this.requestData.responseParams = copyData
        },
        //=====================================其他操作=====================================//
        //将变量转换为实际数据
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
        //生成请求数据
        generateParams(type = "string") {
            return {
                id: this.uuid(),
                key: "",
                description: "",
                type: type,
                value: "",
                required: true,
            }
        },
    }
};
</script>



<style lang="scss">
.response {
    height: calc(100vh - 120px);
    overflow-y: auto;
    .response-wrapper {
        min-height: size(200);
        max-height: size(320);
        overflow-y: auto;
    }
    .baseInfo {
        position: sticky;
        top: 0;
        background: $white;
        z-index: 1;
        box-shadow: $box-shadow-sm;
        padding-bottom: size(10);
    }
    .res-data {
        min-height: size(100);
        max-height: size(300);
    }
    .res-text {
        width: 100%;
        max-height: size(300);
        overflow: auto;
    }
    .pdf-style {
        width: 100%;
        height: size(300);
    }
    .img-style {
        width: size(300);
        height: size(300);
    }
    .office-style {
        font-size: fz(18);
        display: flex;
    }
}
</style>
