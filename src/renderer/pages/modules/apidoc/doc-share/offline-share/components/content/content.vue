/*
    创建者：shuxiaokai
    创建时间：2020-07-06 17:57
    模块名称：文档书写区域区域
    备注：xxxx
*/
<template>
    <div v-if="tabs && tabs.length > 0" class="view-content d-flex" tabindex="0">
        <div v-loading="loading2" :element-loading-text="randomTip()" element-loading-background="rgba(255, 255, 255, 0.9)" class="w-80">
            <!-- 基本配置 -->
            <s-collapse title="基本信息">
                <div class="d-flex">
                    <div class="w-60 flex0">
                        <div class="my-2 d-flex a-center">
                            <span class="flex0">请求地址：</span>
                            <s-ellipsis-content :value="request.url.host + request.url.path" max-width="100%"></s-ellipsis-content>
                        </div>
                        <div class="my-2">
                            <span>请求方式：</span>
                            <span class="green">{{ request.methods.toUpperCase() }}</span>
                        </div>
                    </div>
                    <div>
                        <div class="my-2 d-flex a-center">
                            <span>最近更新日期：</span>
                            <span class="mr-2">{{ new Date(docInfo.updatedAt).toLocaleString() }}</span>
                            <!-- <svg class="svg-icon" aria-hidden="true" @click="dialogVisible4 = true">
                                <use xlink:href="#iconlishi"></use>
                            </svg>  -->
                        </div>  
                    </div>
                </div>
            </s-collapse>
            <s-collapse v-if="0" title="操作">
                <div class="request mb-2">
                    <div class="d-flex a-center">
                        <div>
                            <el-radio-group v-model="request.url.host" size="mini">
                                <el-popover placement="top-start" trigger="hover" :close-delay="0" content="mock">
                                    <el-radio slot="reference" label="__mock__" border>mock</el-radio>
                                </el-popover>
                                <el-popover v-for="(item, index) in hostEnum" :key="index" :close-delay="0" placement="top-start" trigger="hover" :content="item.url">
                                    <el-radio slot="reference" :label="item.url" border>{{ item.name }}</el-radio>
                                </el-popover>
                            </el-radio-group>
                        </div>
                        <el-button v-if="!loading3" type="success" size="mini" @click="sendRequest">发送请求</el-button>
                        <el-button v-if="loading3" type="danger" size="mini" @click="stopRequest">取消请求</el-button>
                        <el-button size="mini" type="primary" @click="dialogVisible2 = true;">变量管理</el-button>
                        <el-button type="primary" size="mini" @click="dialogVisible3 = true;">修改记录</el-button>
                    </div>
                </div>                
            </s-collapse>
            <!-- 请求参数 -->
            <div ref="paramsWrap" class="params-wrap">
                <s-collapse title="请求头">
                    <template v-if="request.header.length > 1">
                        <template v-for="(item, index) in request.header">
                            <div v-if="item.key" class="d-flex a-center mt" :key="index">
                                <span v-if="item.key" class="flex0">{{ item.key }}：</span>
                                <s-ellipsis-content :value="convertVariable(item.value)" :max-width="400" copy></s-ellipsis-content>
                            </div>                    
                        </template>
                    </template>
                    <div v-else class="f-xs gray-500">暂无数据</div>
                </s-collapse>
                <s-collapse title="请求参数">
                    <s-tree-json :data="request.requestParams"></s-tree-json>
                </s-collapse>
                <s-collapse title="响应参数">
                    <s-tree-json :data="request.responseParams" value-width="300px">
                        <template v-slot:operation="{ data }">
                            <span class="cursor-pointer" @click="handleCopyTable(data)">复制为表格</span>
                        </template>
                    </s-tree-json>
                </s-collapse>
            </div>            
        </div>
        <div v-if="0" class="w-35 flex1">
            <s-response :request-data="request"></s-response>
        </div>
        <template v-if="!$root.$data._shareConfig">
            <s-host-manage v-if="dialogVisible" :visible.sync="dialogVisible" @change="getHostEnum"></s-host-manage>
            <s-variable-manage v-if="dialogVisible2" :visible.sync="dialogVisible2" @change="handleVariableChange"></s-variable-manage>
            <s-doc-record-dialog  v-if="dialogVisible3" :visible.sync="dialogVisible3"></s-doc-record-dialog>
            <s-convert-code v-if="dialogVisible4" :visible.sync="dialogVisible4" :code-data="codeData"></s-convert-code>            
        </template>
    </div>
    <div v-else></div>
</template>

<script>
import response from "./components/response"
import hostManage from "./dialog/host-manage"
import variableManage from "./dialog/variable-manage"
import docRecord from "./dialog/doc-record/doc-record"
import convertCode from "./dialog/convert-code"
import { dfsForest } from "@/lib/index"
export default {
    components: {
        "s-host-manage": hostManage,
        "s-response": response,
        "s-variable-manage": variableManage,
        "s-doc-record-dialog": docRecord,
        "s-convert-code": convertCode
    },
    data() {
        return {
            //=====================================请求基本信息====================================//
            request: {
                methods: "get", //---------------请求方式
                requestType: "query", //
                url: {
                    host: "", //-----------------主机(服务器)地址
                    path: "", //-----------------接口路径
                }, //----------------------------请求地址信息
                requestParams: [
                    {
                        id: this.uuid(),
                        key: "", //--------------请求参数键
                        value: "", //------------请求参数值
                        type: "string", //-------------请求参数值类型
                        description: "", //------描述
                        required: true, //-------是否必填
                        children: [], //---------子参数
                    }
                ],
                responseParams: [
                    {
                        id: this.uuid(),
                        key: "", //--------------请求参数键
                        value: "", //------------请求参数值
                        type: "string", //-------------请求参数值类型
                        description: "", //------描述
                        required: true, //-------是否必填
                        children: [], //---------子参数
                    }
                ],
                header: [
                    {
                        id: this.uuid(),
                        key: "", //--------------请求头键
                        value: "", //------------请求头值
                        type: "string", //-------请求头值类型
                        description: "", //------描述
                        required: true, //-------是否必填
                        children: [], //---------子参数
                    }
                ], //----------------------------请求头信息
                description: "在这里输入文档描述", //--------------请求描述
                _description: "", //-------------请求描述拷贝
                _variableChange: true, //----------hack强制触发request数据发生改变
            },
            docInfo: {}, //----------------------文档基本信息
            //=====================================域名相关====================================//
            hostEnum: [], //---------------------域名列表
            //=====================================特殊复制操作====================================//
            codeData: [],
            //=====================================其他参数====================================//
            cancel: [], //-----------------------需要取消的接口
            loading: false, //-------------------保存接口
            loading2: false, //------------------获取文档详情接口
            dialogVisible: false, //-------------域名维护弹窗  
            dialogVisible2: false, //------------全局变量管理弹窗
            dialogVisible3: false, //------------文档修改记录弹窗
            dialogVisible4: false, //------------表格参数转换弹窗
            ready: false, //---------------------是否完成第一次数据请求
        };
    },
    computed: {
        currentSelectDoc() { //当前选中的doc
            return this.$store.state.apidoc.activeDoc[this.$root.$data._shareConfig.id];
        },
        tabs() { //全部tabs
            return this.$store.state.apidoc.tabs[this.$root.$data._shareConfig.id];
        },
        requestParams() {
            const copyData = JSON.parse(JSON.stringify(this.request.requestParams)); //扁平数据拷贝
            dfsForest(copyData, {
                rCondition(value) {
                    return value ? value.children : null;
                },
                rKey: "children",
                hooks: (val, i, forestData, parent) => {
                    if (val && !val._select) {
                        if (!parent) {
                            copyData.splice(i, 1);
                        } else {
                            parent.children.splice(i, 1);
                        }
                    }
                }
            });
            return copyData;
        },
        //全局变量
        variables() {
            return this.$store.state.apidoc.variables || [];
        },
        //发送请求状态
        loading3() {
            return this.$store.state.apidoc.loading;
        },
        publishRecords() {
            if (this.docInfo.publishRecords) {
                return this.docInfo.publishRecords.slice(0).sort((a, b) => {
                    const aTime = new Date(a.time).valueOf();
                    const bTime = new Date(b.time).valueOf();
                    return bTime - aTime;
                })
            } else {
                return []
            }
        }
    },
    watch: {
        currentSelectDoc: {
            handler(val, oldVal) {
                if (val) {
                    if (!oldVal || val._id !== oldVal._id) {
                        this.$store.commit("apidoc/clearRespons");
                        this.getDocDetail();
                    }
                }
            },
            deep: true,
            immediate: true
        }
    },
    mounted() {
        if (!this.$root.$data._shareConfig) {
            this.getHostEnum(); //获取host枚举值
        }
    },
    methods: {
        //=====================================获取数据====================================//
        //获取文档详情
        getDocDetail() {
            if (!this.currentSelectDoc || !this.currentSelectDoc._id) { //没有id不请求数据
                return
            }
            if (window.SHARE_DATA) {
                const docData = window.SHARE_DATA.docs.find(val => {
                    return val._id === this.currentSelectDoc._id
                })
                this.docInfo = docData;
                Object.assign(this.request, docData.item);
            }
        },
        //获取host枚举值
        getHostEnum() {
            const params = {
                projectId: this.$root.$data._shareConfig.id,
            };
            this.axios.get("/api/project/doc_service", { params }).then(res => {
                this.hostEnum = res.data;
            }).catch(err => {
                console.error(err);
            })
        },
        //接口不存在提醒用户，可能是同时操作的用户删掉了这个接口导致接口不存在
        confirmInvalidDoc() {
            this.$confirm("当前接口不存在，可能已经被删除!", "提示", {
                confirmButtonText: "关闭接口",
                cancelButtonText: "取消",
                type: "warning"
            }).then(() => {
                this.$store.commit("apidoc/deleteTabById", {
                    projectId: this.$root.$data._shareConfig.id,
                    deleteIds: [this.currentSelectDoc._id]
                });
                if (!this.tabs.find(val => val._id === this.currentSelectDoc._id)) { //关闭左侧后若在tabs里面无法找到选中节点，则取第一个节点为选中节点
                    this.$store.commit("apidoc/changeCurrentTab", {
                        projectId: this.$root.$data._shareConfig.id,
                        activeNode: this.tabs[this.tabs.length - 1],
                    });
                }
            }).catch(err => {
                if (err === "cancel" || err === "close") {
                    return;
                }
                this.$errorThrow(err, this);
            });
        },
        generateParams() {
            return {
                id: this.uuid(),
                key: "", //--------------请求头键
                value: "", //------------请求头值
                type: "string", //-------请求头值类型
                description: "", //------描述
                required: true, //-------是否必填
                children: [], //---------子参数
            };
        },
        //=====================================发送请求====================================//
        //发送请求
        sendRequest() {
            return new Promise((resolve, reject) => {
                this.$store.commit("apidoc/changeLoading", true);
                const copyRequestInfo =  JSON.parse(JSON.stringify(this.request)); //数据拷贝,防止数据处理过程中改变拷贝请求参数的值
                const requestParams = this.convertPlainParamsToTreeData(copyRequestInfo.requestParams, true); //跳过未选中的参数
                const headerParams = this.convertPlainParamsToTreeData(copyRequestInfo.header);
                const url = copyRequestInfo.url.host + copyRequestInfo.url.path;
                const method = copyRequestInfo.methods.toLowerCase();
                const headers = headerParams;
                const data = requestParams;
                console.log(data, 222)
                this.$store.dispatch("apidoc/sendRequest", { url, method, headers, data }).then(() => {
                    resolve();
                }).catch(err => {
                    console.error(err);
                    reject(err)
                }).finally(() => {
                    this.$store.commit("apidoc/changeLoading", false);
                });                
            })
        },
        //取消请求
        stopRequest() {
            this.loading3 = false;
            this.$refs["response"].stopRequest();
        },
        //=====================================快捷操作====================================//
        //复制为表格数据
        handleCopyTable(data) {
            this.codeData = data.raw;
            this.dialogVisible4 = true;
        },
        //=====================================其他操作=====================================//
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
        //全局变量改变
        handleVariableChange() {
            console.log("change")
            this.request._variableChange = !this.request._variableChange;
        },
        
    }
};
</script>



<style lang="scss">
.view-content {
    padding: size(10) size(0) size(10) size(20);
    .request {
        .view-title {
            padding: size(5) size(10);
            height: size(38);
            border: 1px dashed $gray-500;
        }
        .el-radio {
            margin-right: size(10);
        }
    }
    .params-wrap {
        max-height: calc(100vh - 150px);
        overflow-y: auto;
    }
    .svg-icon {
        width: size(16);
        height: size(16);
        cursor: pointer;
    }
}
</style>
