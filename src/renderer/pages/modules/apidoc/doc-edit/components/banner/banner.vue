/*
    创建者：shuxiaokai
    创建时间：2020-06-24 20:33
    模块名称：文档管理banner导航页面
    备注：xxxx
*/
<template>
    <div class="banner">
        <!-- 工具栏 -->
        <div class="tool">
            <h2 class="gray-700 f-lg text-center text-ellipsis" :title="$route.query.name">{{ $route.query.name }}</h2>
            <el-input v-model="queryData" class="doc-search" placeholder="支持文档名称，文档url搜索" clearable @input="handleSearchTree"></el-input>
            <div class="tool-icon d-flex j-between mt-1 px-1">
                <el-tooltip class="item" effect="dark" content="新增文件夹" :open-delay="300">
                    <svg class="svg-icon" aria-hidden="true" @click="handleOpenAddFolderDialog();docParentId = '';">
                        <use xlink:href="#iconxinzengwenjian"></use>
                    </svg>                        
                </el-tooltip>
                <el-tooltip class="item" effect="dark" content="新增文件" :open-delay="300">
                    <svg class="svg-icon" aria-hidden="true" @click="handleOpenAddFileDialog();docParentId = '';">
                        <use xlink:href="#iconwenjian"></use>
                    </svg>               
                </el-tooltip>
                <el-tooltip class="item" effect="dark" content="历史记录" :open-delay="300">
                    <svg class="svg-icon" aria-hidden="true" @click="dialogVisible4 = true">
                        <use xlink:href="#iconlishi"></use>
                    </svg>               
                </el-tooltip>
                <svg class="item svg-icon" aria-hidden="true" @click="freshBanner">
                    <use xlink:href="#iconshuaxin"></use>
                </svg>
                <el-dropdown trigger="click" class="mr-1">
                    <i class="more-op el-icon-more" title="更多操作"></i>
                    <el-dropdown-menu slot="dropdown">
                        <el-dropdown-item @click.native="handleViewDoc">预览文档</el-dropdown-item>
                        <el-dropdown-item @click.native="dialogVisible6 = true">导出文档</el-dropdown-item>
                        <el-dropdown-item @click.native="dialogVisible3 = true">导入文档</el-dropdown-item>
                    </el-dropdown-menu>
                </el-dropdown>
                <!-- <el-tooltip class="item" effect="dark" content="历史记录" :open-delay="300">
                    <svg class="svg-icon" aria-hidden="true" @click="dialogVisible4 = true">
                        <use xlink:href="#iconlishi"></use>
                    </svg>               
                </el-tooltip>
                <el-tooltip class="item" effect="dark" content="预览文档" :open-delay="300">
                    <svg class="svg-icon" aria-hidden="true" @click="handleViewDoc">
                        <use xlink:href="#iconpreview"></use>
                    </svg>               
                </el-tooltip>
                <el-tooltip class="item" effect="dark" content="导出为word" :open-delay="300">
                    <s-download url="/api/project/doc_word" :params="{ projectId: $route.query._id }">
                        <svg class="svg-icon" aria-hidden="true">
                            <use xlink:href="#icondaochu"></use>
                        </svg>                    
                    </s-download>
                </el-tooltip>
                <el-tooltip class="item" effect="dark" content="导入" :open-delay="300">
                    <svg class="svg-icon" aria-hidden="true" @click="dialogVisible3 = true">
                        <use xlink:href="#icondaoru"></use>
                    </svg>
                </el-tooltip> -->
                
            </div>
        </div>
        <!-- 树形文档导航 -->
        <div v-loading="loading" :element-loading-text="randomTip()" element-loading-background="rgba(255, 255, 255, 0.9)" class="doc-nav">
            <el-tree 
                    ref="docTree"
                    :data="navTreeData" 
                    node-key="_id" 
                    empty-text="点击按钮新增文档"
                    :default-expanded-keys="defaultExpandedKeys"
                    :expand-on-click-node="true" 
                    :draggable="enableDrag"
                    :allow-drop="handleCheckNodeCouldDrop"
                    :filter-node-method="filterNode"
                    @node-contextmenu="handleContextmenu"
                    @node-drop="handleNodeDropSuccess"
                    @node-expand="clearContextmenu"
                    @node-collapse="clearContextmenu"
                    @node-click="handleNodeClick" 
            >
                <template slot-scope="scope">
                    <div 
                            class="custom-tree-node"
                            :class="{'selected': multiSelectNode.find(val => val.data._id === scope.data._id), 'active': currentSelectDoc && currentSelectDoc._id === scope.data._id}"
                            tabindex="1"
                            @keydown="handleKeydown($event, scope.data)"
                            @keyup="handleKeyUp"
                            @click="handleClickNode($event, scope)"
                            @mouseover="hoverNodeId = scope.data._id"
                            @mouseout="hoverNodeId = ''"
                    >
                        <!-- file渲染 -->
                        <template v-if="!scope.data.isFolder">
                            <template v-for="(req) in validRequestMethods">
                                <span v-if="scope.data.item.methods === req.value.toLowerCase()" :key="req.name" class="label" :style="{color: req.iconColor}">{{ req.name.toLowerCase() }}</span>
                            </template>  
                            <!-- <span v-if="scope.data.item.methods === 'get'" class="label green">get</span>
                            <span v-else-if="scope.data.item.methods === 'post'" class="label yellow">post</span>
                            <span v-else-if="scope.data.item.methods === 'put'" class="label blue">put</span>
                            <span v-else-if="scope.data.item.methods === 'delete'" class="label red">del</span>  
                            <img v-else :src="require('@/assets/imgs/apidoc/file.png')" width="16px" height="16px"/>  -->
                            <s-emphasize v-if="renameNodeId !== scope.data._id" :title="scope.data.docName" :value="scope.data.docName" :keyword="queryData" class="node-name text-ellipsis ml-1"></s-emphasize>
                            <!-- <span v-if="renameNodeId !== scope.data._id" :title="scope.data.docName" class="node-name text-ellipsis ml-1">{{ scope.data.docName }}</span> -->
                            <input v-else v-model="scope.data.docName" placeholder="不能为空" type="text" class="rename-ipt f-sm ml-1" @blur="handleChangeNodeName(scope.data)" @keydown.enter="handleChangeNodeName(scope.data)">
                            <el-dropdown 
                                    v-show="hoverNodeId === scope.data._id"
                                    class="node-more ml-auto mr-2"
                                    trigger="click"
                                    @command="(command) => { handleSelectDropdown(command, scope.data, scope.node) }"
                                    @click.native.stop="() =>{}"
                            >
                                <span class="el-icon-more"></span>
                                <el-dropdown-menu slot="dropdown">
                                    <el-dropdown-item v-if="scope.data.isFolder" command="addFile">新建文档</el-dropdown-item>
                                    <el-dropdown-item v-if="scope.data.isFolder" command="addByTemplate">以模板新建</el-dropdown-item>
                                    <el-dropdown-item v-if="!scope.data.isFolder" command="copy">复制接口</el-dropdown-item>
                                    <el-dropdown-item command="rename">重命名</el-dropdown-item>
                                    <el-dropdown-item command="delete">删除</el-dropdown-item>
                                </el-dropdown-menu>
                            </el-dropdown>  
                        </template>
                        <!-- 文件夹渲染 -->
                        <template v-if="scope.data.isFolder">
                            <img :src="require('@/assets/imgs/apidoc/folder.png')" width="16px" height="16px"/>    
                            <span v-if="renameNodeId !== scope.data._id" :title="scope.data.docName" class="node-name text-ellipsis ml-1">{{ scope.data.docName }}</span>
                            <input v-else v-model="scope.data.docName" placeholder="不能为空" type="text" class="rename-ipt f-sm ml-1" @blur="handleChangeNodeName(scope.data)" @keydown.enter="handleChangeNodeName(scope.data)">
                            <el-dropdown 
                                    v-show="hoverNodeId === scope.data._id"
                                    class="node-more ml-auto mr-2"
                                    trigger="click"
                                    @command="(command) => { handleSelectDropdown(command, scope.data, scope.node) }"
                                    @click.native.stop="() =>{}"
                            >
                                <span class="el-icon-more"></span>
                                <el-dropdown-menu slot="dropdown">
                                    <el-dropdown-item v-if="scope.data.isFolder" command="addFile">新建文档</el-dropdown-item>
                                    <el-dropdown-item v-if="scope.data.isFolder" command="addFolder">新建文件夹</el-dropdown-item>
                                    <el-dropdown-item v-if="scope.data.isFolder" command="addByTemplate">以模板新建</el-dropdown-item>
                                    <el-dropdown-item command="rename">重命名</el-dropdown-item>
                                    <el-dropdown-item command="delete">删除</el-dropdown-item>
                                </el-dropdown-menu>
                            </el-dropdown>                      
                        </template>
                    </div>
                </template>
            </el-tree>
        </div>
        <!-- 弹窗 -->
        <s-add-folder-dialog v-if="dialogVisible" :visible.sync="dialogVisible" :pid="docParentId" @success="handleAddFileAndFolderCb"></s-add-folder-dialog>
        <s-add-file-dialog v-if="dialogVisible2" :visible.sync="dialogVisible2" :pid="docParentId" @success="handleAddFileAndFolderCb"></s-add-file-dialog>
        <s-import-doc-dialog v-if="dialogVisible3" :visible.sync="dialogVisible3" @success="init"></s-import-doc-dialog>
        <s-history-dialog :visible.sync="dialogVisible4"></s-history-dialog>
        <s-template-dialog :visible.sync="dialogVisible5"></s-template-dialog>
        <s-export-dialog :visible.sync="dialogVisible6"></s-export-dialog>
    </div>
</template>

<script>
import Vue from "vue"
import { findoNode, forEachForest, findPreviousSibling, findNextSibling, findParentNode, debounce } from "@/lib/index"
import addFolderDialog from "../../dialog/add-folder"
import addFileDialog from "../../dialog/add-file"
import importDoc from "../../dialog/import-doc"
import historyDialog from "./dialog/history"
import templateDialog from "./dialog/template"
import exportDialog from "./dialog/export"
import contextmenu from "./components/contextmenu"
export default {
    components: {
        "s-add-folder-dialog": addFolderDialog,
        "s-add-file-dialog": addFileDialog,
        "s-import-doc-dialog": importDoc,
        "s-history-dialog": historyDialog,
        "s-template-dialog": templateDialog,
        "s-export-dialog": exportDialog,
    },
    computed: {
        navTreeData() { //-------树形导航数据
            return this.$store.state.apidoc.banner;
        },
        tabs() { //--------------全部tabs
            return this.$store.state.apidoc.tabs[this.$route.query.id];
        },
        currentSelectDoc() { //--当前选中的文档
            return this.$store.state.apidoc.activeDoc[this.$route.query.id];
        },
        docRules() { //---------文档规则
            return this.$store.state.apidocRules;
        },
        validRequestMethods() {
            return this.$store.state.apidocRules.requestMethods.filter(val => val.enabled);
        }
    },
    watch: {
        currentSelectDoc: {
            handler(val) {
                if (val && val._id) {
                    this.defaultExpandedKeys.splice(0, 1, val._id);
                }
            },
            deep: true
        }
    },
    data() {
        return {
            //=====================================文档增删改查====================================//
            queryData: "", //------------文档过滤条件
            docParentId: "", //----------文档父id
            contextmenu: null, //--------右键弹窗
            renameNodeId: "", //---------正在重命名的节点
            pressCtrl: false, //---------是否按住ctrl键
            multiSelectNode: [], //------按住ctrl+鼠标左键多选节点
            enableDrag: true, //---------是否允许文档被拖拽
            defaultExpandedKeys: [], //--默认展开的文档key值
            //=====================================其他参数====================================//
            hoverNodeId: "", //----------控制导航节点更多选项显示
            dialogVisible: false, //-----新增文件夹弹窗
            dialogVisible2: false, //----新增文件弹窗
            dialogVisible3: false, //----导入第三方文档弹窗
            dialogVisible4: false, //----查看历史记录
            dialogVisible5: false, //----以模板新建
            dialogVisible6: false, //----导出文档
            loading: false, //-----------左侧树形导航加载
        };
    },
    mounted() {
        this.init();
    },
    methods: {
        //=====================================初始化相关====================================//
        init() {
            this.getDocBanner();
            document.documentElement.addEventListener("click", () => {
                // e.stopPropagation();
                this.clearContextmenu();
                this.multiSelectNode = [];
            })
        },
        //刷新banner
        freshBanner() {
            if (!this.loading) {
                this.getDocBanner();
            }
        },
        getDocBanner() {
            this.loading = true;
            this.$store.dispatch("apidoc/getDocBanner", { _id: this.$route.query.id }).then(() => {
                this.loading = false;
            });
        },
        //=====================================导航操作==================================//
        //文档下拉框选择 重命名，删除，新增...
        handleSelectDropdown(command, data, node) {
            /*eslint-disable indent*/
            switch (command) {
                case "addFile":
                    this.docParentId = data._id;
                    if (node && node.childNodes.length >= this.docRules.fileInFolderLimit) {
                        this.$message.warning(`单个文件夹里面文档个数不超过${this.docRules.fileInFolderLimit}个`);
                    } else {
                        this.handleOpenAddFileDialog();
                    }
                    break;
                case "addFolder":
                    this.docParentId = data._id;
                    this.handleOpenAddFolderDialog();
                    break;
                case "rename":
                    this.$set(data, "_docName", data.docName); //文档名称备份,防止修改名称用户名称填空导致异常
                    this.renameNodeId = data._id;
                    this.$nextTick(() => {
                        document.querySelector(".rename-ipt").focus();
                        this.enableDrag = false;                    
                    })
                    break;
                case "delete":
                    this.handleDeleteItem(data, node);
                    break;
                case "addByTemplate":
                    this.addByTemplate(data);
                    break;
                case "copy":
                    if (node && node.parent && node.parent.childNodes && node.parent.childNodes.length >= this.docRules.fileInFolderLimit) {
                        this.$message.warning(`单个文件夹里面文档个数不超过${this.docRules.fileInFolderLimit}个`);
                    } else {
                        this.copyDoc(data, node);
                    }
                    break;
                default:
                    break;
            }
        },
        //创建鼠标右键dom元素
        handleContextmenu(e, data, node) {
            e.stopPropagation();
            this.clearContextmenu(); //清除contextmenu
            const ContextmenuConstructor = Vue.extend(contextmenu);
            const x = e.clientX; //当前点击位置
            const y = e.clientY; //当前点击位置
            let operations = [];
            if (this.multiSelectNode.length > 0) {
                operations = ["deleteMany"];
            } else {
                operations = data.isFolder ? ["file", "folder", "template", "rename", "delete"] : ["rename", "delete", "copy"];
            }
            this.contextmenu = new ContextmenuConstructor({
                propsData: {
                    operations,
                    left: x,
                    top: y
                },
            }).$mount();
            document.body.appendChild(this.contextmenu.$el);
            this.contextmenu.$on("file", () => {
                this.docParentId = data._id;
                if (node && node.childNodes.length >= this.docRules.fileInFolderLimit) {
                    this.$message.warning(`单个文件夹里面文档个数不超过${this.docRules.fileInFolderLimit}个`);
                } else {
                    this.handleOpenAddFileDialog();
                }
            })
            this.contextmenu.$on("folder", () => {
                this.docParentId = data._id;
                this.handleOpenAddFolderDialog();
            })
            this.contextmenu.$on("rename", () => {
                this.$set(data, "_docName", data.docName); //文档名称备份,防止修改名称用户名称填空导致异常
                this.renameNodeId = data._id;
                this.$nextTick(() => {
                    document.querySelector(".rename-ipt").focus();
                    this.enableDrag = false;                    
                })
            })
            this.contextmenu.$on("delete", () => {
                this.handleDeleteItem(data, node);
            })
            this.contextmenu.$on("deleteMany", () => {
                this.handleDeleteManyItem();
            })
            this.contextmenu.$on("template", () => {
                this.addByTemplate(data);
            })
            this.contextmenu.$on("copy", () => {
                if (node && node.parent && node.parent.childNodes && node.parent.childNodes.length >= this.docRules.fileInFolderLimit) {
                    this.$message.warning(`单个文件夹里面文档个数不超过${this.docRules.fileInFolderLimit}个`);
                } else {
                    this.copyDoc(data, node);
                }
            })
        },   
        //处理节点上面keydown快捷方式(例如f2重命名)
        handleKeydown(e, data) {
            if (e.code === "F2") {
                this.$set(data, "_docName", data.docName); //文档名称备份,防止修改名称用户名称填空导致异常
                this.renameNodeId = data._id;
                this.$nextTick(() => {
                    document.querySelector(".rename-ipt").focus();
                    this.enableDrag = false;                    
                })
            } else if (e.code === "ControlLeft" || e.code === "ControlRight") {
                this.pressCtrl = true;
            }
        }, 
        //按键放开
        handleKeyUp() {
            this.pressCtrl = false;
        },
        //点击节点
        handleClickNode(e, { node },) {
            if (this.pressCtrl) {
                e.stopPropagation();
                const delIndex = this.multiSelectNode.findIndex(val => val._id === node.data._id);
                if (delIndex !== -1) {
                    this.multiSelectNode.splice(delIndex, 1);
                } else {
                    this.multiSelectNode.push(node);
                }
            }
        },
        //添加文件夹或文档成功回调函数
        handleAddFileAndFolderCb(data) {
            const pNode = findoNode(this.docParentId, this.navTreeData, null, { id: "_id" });
            if (!pNode) { //插入到根元素
                if (data.isFolder) { //如果是文件夹则放在第一位
                    let folderIndex = -1;
                    for (let i = 0,len = this.navTreeData.length; i < len; i++) {
                        if (!this.navTreeData[i].isFolder) {
                            this.navTreeData.splice(i, 0, data);
                            folderIndex = i;
                            break;
                        }
                    }
                    if (folderIndex === -1) { //不存在文件则直接添加到末尾
                        this.navTreeData.push(data);
                    }
                } else { //如果是文本
                    this.navTreeData.push(data);
                }
            } else { //插入到文件夹里面
                if (!pNode.children) {
                    this.$set(pNode, "children", [])
                }
                if (data.isFolder) { //如果是文件夹则放在第一位
                    this.defaultExpandedKeys.push(data._id)
                    let folderIndex = -1;
                    for (let i = 0,len = pNode.children.length; i < len; i++) {
                        if (!pNode.children[i].isFolder) {
                            pNode.children.splice(i, 0, data);
                            folderIndex = i;
                            break;
                        }
                    }
                    if (folderIndex === -1) { //不存在文件则直接添加到末尾
                        pNode.children.push(data);
                    }
                } else { //如果是文本
                    pNode.children.push(data);
                }
            }
            if (!data.isFolder) { //文件夹不做处理
                data.tabType = "doc"
                this.$store.commit("apidoc/addTab", data);
                this.$store.commit("apidoc/changeCurrentTab", {
                    projectId: this.$route.query.id,
                    activeNode: data
                });
            }
        },
        //判断是否允许拖拽
        handleCheckNodeCouldDrop(draggingNode, dropNode, type) {
            if (!draggingNode.data.isFolder && dropNode.data.isFolder && type !== "inner") { //不允许文件在文件夹前面
                return type !== "prev";
            }
            if (draggingNode.data.isFolder && !dropNode.data.isFolder) {
                return false;
            } else if (!dropNode.data.isFolder) { 
                return type !== "inner";
            } else {
                return true
            }
        },
        //拖拽成功时候触发
        handleNodeDropSuccess(node, dropNode, type) {
            const params = {
                _id: node.data._id, //当前节点id
                pid: "", //父元素
                sort: 0, //当前节点排序效果
            };
            const dragNodeParentId = findParentNode(node.data._id, this.navTreeData, null, {id: "_id"});
            const dropNodeParentId = findParentNode(dropNode.data._id, this.navTreeData, null, {id: "_id"})
            console.log(dragNodeParentId, dropNodeParentId)
            let pData = null;
            pData = findParentNode(node.data._id, this.navTreeData, null, {id: "_id"});
            params.pid = pData ? pData._id : "";
            // if ((node.level !== dropNode.level) || (node.level === dropNode.level && type === "inner")) { //将节点放入子节点中
            //     pData = findParentNode(node.data._id, this.navTreeData, null, {id: "_id"});
            //     params.pid = pData ? pData._id : "";
            //     while (pData != null) {
            //         pData = findParentNode(pData._id, this.navTreeData, null, {id: "_id"});
            //     }
            // } else if (node.level === dropNode.level && type !== "inner") {
            //     params.pid = node.data.pid;
            //     pData = findParentNode(node.data._id, this.navTreeData, null, {id: "_id"});
            //     while (pData != null) {
            //         pData = findParentNode(pData._id, this.navTreeData, null, {id: "_id"});
            //     }
            // }
            
            if (type === "inner") {
                params.sort = Date.now();
            } else {
                const nextSibling = findNextSibling(node.data._id, this.navTreeData, null, { id: "_id" }) || {};
                const previousSibling = findPreviousSibling(node.data._id, this.navTreeData, null, { id: "_id" }) || {};
                const previousSiblingSort = previousSibling.sort || 0;
                const nextSiblingSort = nextSibling.sort || Date.now();
                params.sort = (nextSiblingSort + previousSiblingSort) / 2;                
                node.data.sort = (nextSiblingSort + previousSiblingSort) / 2;
            }
            this.axios.put("/api/project/change_doc_pos", params).then(() => {
                
            }).catch(err => {
                this.$errorThrow(err, this);
            });
        },
        //点击节点
        handleNodeClick(data, node) {
            if (!node.data.isFolder) { //文件夹不做处理
                node.data.tabType = "doc"
                this.$store.commit("apidoc/addTab", node.data);
                this.$store.commit("apidoc/changeCurrentTab", {
                    projectId: this.$route.query.id,
                    activeNode: node.data
                });
            }
            this.clearContextmenu();
            this.multiSelectNode = [];

        },
        //拷贝节点
        copyDoc(data, node) {
            const params = {
                _id: data._id
            };
            this.axios.post("/api/project/copy_doc", params).then(res => {
                const pNode = node.parent;
                if (pNode.level === 0) { //在根元素下面插入
                    pNode.data.push(res.data);
                } else { //在某个元素下面插入
                    pNode.data.children.push(res.data);
                }
                if (!res.data.isFolder) { //文件夹不做处理
                    res.data.tabType = "doc"
                    this.$store.commit("apidoc/addTab", res.data);
                    this.$store.commit("apidoc/changeCurrentTab", {
                        projectId: this.$route.query.id,
                        activeNode: res.data
                    });
                }
            }).catch(err => {
                this.$errorThrow(err, this);
            });
        },
        //=====================================前后端交互====================================//
        handleSearchTree() {
            this.search();
        },
        search: debounce(function() {
            this.searchResult = [];
            const params = {
                projectId: this.$route.query.id,
                url: this.queryData.trim()
            };
            this.axios.get("/api/project/filter_doc", { params }).then(res => {
                if (res.data.length === 0) {
                    this.defaultExpandedKeys = [];
                    this.searchResult = [];
                } else {
                    this.defaultExpandedKeys = Array.from(new Set(this.defaultExpandedKeys.concat(res.data.map(val => val._id))))
                    this.searchResult = Array.from(new Set(this.searchResult.concat(res.data.map(val => val))));                    
                }
                this.$refs.docTree.filter();
            }).catch(err => {
                this.$errorThrow(err, this);
            }).finally(() => {
                this.loading = false;
            });
        }),
        filterNode(value, data) {
            const matchName = !!this.searchResult.find(val => val.docName === data.label);
            const matchUrl = !!this.searchResult.find(val => val._id === data._id);
            const matchAll = this.queryData.trim() === "";
            return matchName || matchUrl || matchAll;
        },
        //删除某一项
        handleDeleteItem(data, node) {
            let deleteId = [];
            deleteId.push(data._id); //删除自己
            if (data.isFolder) { //删除所有子元素
                forEachForest(data.children, (item) => {
                    deleteId.push(item._id);
                });
            } 
            this.$confirm(`此操作将永久删除 ${data.docName} 节点, 是否继续?`, "提示", {
                confirmButtonText: "确定",
                cancelButtonText: "取消",
                type: "warning"
            }).then(() => {
                this.axios.delete("/api/project/doc", { data: { projectId: this.$route.query.id, ids: deleteId }}).then(() => {
                    const pNode = node.parent;
                    if (pNode && pNode.level !== 0) {
                        const nodeIndex = pNode.data.children.findIndex(val => val._id === data._id);
                        pNode.data.children.splice(nodeIndex, 1)
                    } else {
                        const nodeIndex = this.navTreeData.findIndex(val => val._id === data._id);
                        this.navTreeData.splice(nodeIndex, 1);
                    }
                    this.handleDeleteTabsById(deleteId);
                }).catch(err => {
                    this.$errorThrow(err, this);
                });            
            }).catch((err) => {
                if (err === "cancel" || err === "close") {
                    return;
                }
                this.$errorThrow(err, this);
            });
        },
        //批量删除
        handleDeleteManyItem() {
            let deleteId = [];
            const selectNodeCopy = this.multiSelectNode; //点击节点会清空选中数据
            this.multiSelectNode.forEach(val => {
                deleteId.push(val.data._id);
                if (val.data.isFolder) { //删除所有子元素
                    forEachForest(val.data.children || [], (item) => {
                        deleteId.push(item._id);
                    });
                }                 
            })
            this.$confirm(`确认删除选中的${deleteId.length}个节点, 是否继续?`, "提示", {
                confirmButtonText: "确定",
                cancelButtonText: "取消",
                type: "warning"
            }).then(() => {
                this.axios.delete("/api/project/doc", { data: { projectId: this.$route.query.id, ids: deleteId }}).then(() => {
                    selectNodeCopy.forEach(delNode => {
                        const pNode = delNode.parent;
                        if (pNode && pNode.level !== 0) { //非根元素
                            const nodeIndex = pNode.data.children.findIndex(val => val._id === delNode.data._id);
                            pNode.data.children.splice(nodeIndex, 1)
                        } else { //根元素
                            const nodeIndex = this.navTreeData.findIndex(val => val._id === delNode.data._id);
                            this.navTreeData.splice(nodeIndex, 1);
                        }
                        this.handleDeleteTabsById(deleteId);
                    })
                }).catch(err => {
                    this.$errorThrow(err, this);
                });            
            }).catch((err) => {
                if (err === "cancel" || err === "close") {
                    return;
                }
                this.$errorThrow(err, this);
            });
        },
        //根据id删除tab
        handleDeleteTabsById(deleteIds) {
            this.$store.commit("apidoc/deleteTabById", {
                projectId: this.$route.query.id,
                deleteIds: deleteIds
            });
            if (!this.tabs.find(val => val._id === this.currentSelectDoc._id)) { //关闭左侧后若在tabs里面无法找到选中节点，则取第一个节点为选中节点
                this.$store.commit("apidoc/changeCurrentTab", {
                    projectId: this.$route.query.id,
                    activeNode: this.tabs[this.tabs.length - 1],
                });
            }
        },
        //重命名某个节点
        handleChangeNodeName(data) {
            this.renameNodeId = "";
            this.enableDrag = true; //改名以后允许节点拖拽
            if (data.docName.trim() === "") {
                data.docName = data._docName;
                return;
            }
            if (data.docName === data._docName) {
                return;
            }
            const params = {
                _id: data._id,
                docName: data.docName
            };
            this.axios.put("/api/project/change_doc_info", params).then(() => {
                this.$store.commit("apidoc/changeTabInfoById", {
                    _id: data._id,
                    projectId: this.$route.query.id,
                    docName: data.docName
                });
                if (this.currentSelectDoc._id === data._id) {
                    this.$store.commit("apidoc/changeCurrentTabById", {
                        projectId: this.$route.query.id,
                        docName: data.docName
                    });
                }
            }).catch(err => {
                data.docName = data._docName; //修改出错后回复文档名称
                this.$errorThrow(err, this);
            });
            this.renameNodeId = "";
        },
        //=====================================弹窗相关====================================//  
        //打开文件新增弹窗
        handleOpenAddFolderDialog() {
            this.dialogVisible = true;
        },
        //打开文件新增弹窗
        handleOpenAddFileDialog() {
            this.dialogVisible2 = true;
        },
        //预览文档
        handleViewDoc() {
            this.$router.push({
                path: "/v1/apidoc/doc-view",
                query: {
                    id: this.$route.query.id,
                    name: this.$route.query.name
                }
            });
        },
        //以模板新增
        addByTemplate() {
            this.dialogVisible5 = true;
        },

        //=====================================其他操作=====================================//
        //清除contextmenu
        clearContextmenu() {
            if (this.contextmenu) {
                document.body.removeChild(this.contextmenu.$el);
                this.contextmenu = null;
            }  
        },
    }
};
</script>



<style lang="scss">
.banner {
    width: size(300);
    height: 100%;
    border-right: 1px solid $gray-400;
    display: flex;
    flex-direction: column;
    .el-tree-node__content {
        height: size(30);
    }
    .el-tree-node__content:hover {
        background: none;
    }
    .tool {
        position: relative;
        padding: 0 size(20);
        height: size(150);
        background: $gray-200;
        // 搜索框样式
        .doc-search {
            border-radius: 20px;
            .el-input__inner {
                border-radius: 20px;
            }
        }        
        // 快捷方式样式
        .tool-icon {
            position: relative;
            .svg-icon {
                width: size(25);
                height: size(25);
                padding: size(5);
                cursor: pointer;
                &:hover {
                    background: $gray-400;
                }
            }            
        }
        .more-op {
            width: size(25);
            height: size(25);
            line-height: size(25);
            text-align: center;
            cursor: pointer;
            &:hover {
                background: $gray-400;
            }
        }
    }
    .doc-nav {
        height: calc(100vh - #{size(60)} - #{size(150)});
        overflow: auto;
        .custom-tree-node {
            display: flex;
            align-items: center;
            height: 30px;
            width: 100%;
            &:hover {
                background: mix($theme-color, $white, 25%);
            }
            &.active {
                background: mix($theme-color, $white, 25%);
            }
            //selected放在后面覆盖掉active样式
            &.selected {
                background: mix($theme-color, $white, 50%);
            }
            .label {
                display: inline-block;
                width: 25px;
            }
            .node-name {
                display: inline-block;
                max-width: 180px;
                border: 2px solid transparent;
            }
        }
    }

}
</style>
