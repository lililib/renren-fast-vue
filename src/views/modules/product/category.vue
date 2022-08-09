<template>
  <div>
    <el-switch
      v-model="draggable"
      active-text="开启拖拽"
      inactive-text="关闭拖拽"
    >
    </el-switch>
    <el-button v-if="draggable" @click="saveBatch">批量保存</el-button>
    <el-button type="danger" @click="batchDel">批量删除</el-button>
    <el-tree
      :data="menus"
      :props="defaultProps"
      :expand-on-click-node="false"
      show-checkbox
      node-key="catId"
      :default-expanded-keys="expendKeys"
      :draggable="draggable"
      :allow-drop="allowDrop"
      @node-drop="handleDrop"
      ref="menuTree"
    >
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            v-if="node.level <= 2"
            type="text"
            size="mini"
            @click="() => append(data)"
          >
            Append
          </el-button>

          <el-button type="text" size="mini" @click="edit(data)">
            edit
          </el-button>

          <el-button
            v-if="node.childNodes.length == 0"
            type="text"
            size="mini"
            @click="() => remove(node, data)"
          >
            Delete
          </el-button>
        </span>
      </span>
    </el-tree>
    <el-dialog
      :title="title"
      :visible.sync="dialogVisible"
      width="30%"
      close-on-click-modal="false"
    >
      <el-form :model="category">
        <el-form-item label="添加分类">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>

        <el-form-item label="添加图标">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>

        <el-form-item label="添加计量单位">
          <el-input
            v-model="category.productUnit"
            autocomplete="off"
          ></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="submitDate">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
//这里可以导入其他文件（比如：组件，工具 js，第三方插件 js，json

//例如：import 《组件名称》 from '《组件路径》';

export default {
  //import 引入的组件需要注入到对象中才能使用
  components: {},
  props: {},

  data() {
    return {
      pCid: [],
      sublings: [],
      maxLevel: 0,
      title: "",
      submitType: "",
      category: {
        name: "",
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        catId: null,
        icon: "",
        productUnit: "",
      },
      draggable: false,
      dialogVisible: false,
      menus: [],
      expendKeys: [],
      defaultProps: {
        children: "child",
        label: "name",
      },
    };
  },
  methods: {
    batchDel() {
      let del = [];
      let delName=[];
      let checkNodes = this.$refs.menuTree.getCheckedNodes();
      console.log("checkNodes", checkNodes);
      for (let i = 0; i < checkNodes.length; i++) {
        del.push(checkNodes[i].catId);
        delName.push(checkNodes[i].name);
      }
      this.$confirm(`此操作将永久删除【${delName}】菜单, 是否继续?`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(del, false),
          }).then(({ data }) => {
            this.$message({
              message: "删除成功",
              type: "success",
            });
            this.getMenus();
          
          });
        })
        .catch(() => {
          this.$message({
            type: "info",
            message: "已取消删除",
          });
        });
    },

    saveBatch() {
      this.$http({
        url: this.$http.adornUrl("/product/category/update/sort"),
        method: "post",
        data: this.$http.adornData(this.sublings, false),
      }).then(({ data }) => {
        this.$message({
          message: "菜单修改排序等成功",
          type: "success",
        });
        this.getMenus();
        this.expendKeys = this.pCid;
        this.sublings = [];
        this.maxLevel = 0;
        //this.pCid=0;
      });
    },
    handleDrop(draggingNode, dropNode, dropType, ev) {
      let pCid = 0;
      let sublings = null;
      console.log("handleDrop", draggingNode, dropNode, dropType);
      if (dropNode == "inner") {
        pCid = dropNode.data.catId;
        sublings = dropNode.childNodes;
      } else {
        pCid =
          dropNode.parent.data.catId == undefined
            ? 0
            : dropNode.parent.data.catId;
        sublings = dropNode.parent.childNodes;
      }
      this.pCid.push(pCid);
      for (let i = 0; i < sublings.length; i++) {
        if (sublings[i].data.catId == draggingNode.data.catId) {
          let catLevel = draggingNode.level;
          if (sublings[i].level != draggingNode.level) {
            //拖拽节点的层级改变了
            catLevel = sublings[i].level;
            this.updateChileNodeLevel(sublings[i]);
          }
          this.sublings.push({
            catId: sublings[i].data.catId,
            sort: i,
            parentCid: pCid,
            catLevel: catLevel,
          });
        } else {
          this.sublings.push({ catId: sublings[i].data.catId, sort: i });
        }
      }
      console.log("updatesublings", this.sublings);
    },

    updateChileNodeLevel(node) {
      if (node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          var cNode = node.childNodes[i].data;
          this.sublings.push({
            catId: cNode.catId,
            catLevel: node.childNodes[i].level,
          });
          this.updateChileNodeLevel(node.childNodes[i]);
        }
      }
    },
    allowDrop(draggingNode, dropNode, type) {
      console.log("allowDrop", draggingNode, dropNode, type);
      this.countNodeLevel(draggingNode);

      let deep = Math.abs(this.maxLevel - draggingNode.level) + 1;
      console.log("deep", deep, this.maxLevel);
      if (type == "inner") {
        return deep + dropNode.level <= 3;
      } else {
        return deep + dropNode.parent.level <= 3;
      }
    },
    countNodeLevel(node) {
      if (node.childNodes != null && node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          if (node.childNodes[i].level > this.maxLevel) {
            this.maxLevel = node.childNodes[i].level;
          }
          this.countNodeLevel(node.childNodes[i]);
        }
      }
    },

    submitDate() {
      if (this.submitType == "add") {
        this.addCategory();
      }
      if (this.submitType == "edit") {
        this.editCategory();
      }
    },

    getMenus() {
      this.$http({
        url: this.$http.adornUrl("/product/category/list/tree"),
        method: "get",
      }).then(({ data }) => {
        console.log("成功获取到数据", data);
        this.menus = data.data;
      });
    },

    edit(data) {
      this.submitType = "edit";
      this.title = "修改分类";
      console.log("edit", data);
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: "get",
      }).then(({ data }) => {
        console.log("要回显的数据", data);
        this.dialogVisible = true;
        this.category.catId = data.data.catId;
        this.category.name = data.data.name;
        this.category.parentCid = data.data.parentCid;
        this.category.icon = data.data.icon;
        this.category.productUnit = data.data.productUnit;
      });
    },

    append(data) {
      this.dialogVisible = true;
      this.submitType = "add";
      this.title = "添加分类";

      this.category.parentCid = data.catId;
      this.category.catLevel = data.catLevel * 1 + 1;
      this.category.parentCid = data.parentCid;
      this.category.name = "";
      this.category.icon = "";
      this.category.productUnit = "";

      this.category.catId = null;

      this.category.showStatus = 1;
      this.category.sort = 0;

      console.log("append", data);
    },
    editCategory() {
      console.log("进入编辑提交程序");
      var { icon, name, catId, productUnit } = this.category; //数据的解构
      console.log("结构数据", icon, name, catId, productUnit);
      this.$http({
        url: this.$http.adornUrl("/product/category/update"),
        method: "post",
        data: this.$http.adornData({ icon, name, catId, productUnit }, false),
      }).then(({ data }) => {
        this.$message({
          message: "修改成功",
          type: "success",
        });
        this.dialogVisible = false;
        this.getMenus();
        this.expendKeys = [this.category.parentCid];
      });
    },

    addCategory() {
      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: "post",
        data: this.$http.adornData(this.category, false),
      }).then(({ data }) => {
        this.$message({
          message: "添加成功",
          type: "success",
        });
        this.dialogVisible = false;
        this.getMenus();
        this.expendKeys = [this.category.parentCid];
      });
      console.log("add ok", this.category);
    },
    remove(node, data) {
      var ids = [data.catId];
      this.$confirm(`此操作将永久删除【${data.name}】菜单, 是否继续?`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(ids, false),
          }).then(({ data }) => {
            this.$message({
              message: "删除成功",
              type: "success",
            });
            this.getMenus();
            this.expendKeys = [node.parent.data.catId];
          });
        })
        .catch(() => {
          this.$message({
            type: "info",
            message: "已取消删除",
          });
        });

      console.log("remove", node, data);
    },
  },

  //计算属性 类似于 data 概念
  computed: {},
  //监控 data 中的数据变化
  watch: {},
  //方法集合

  //生命周期 - 创建完成（可以访问当前 this 实例）
  created() {
    this.getMenus();
  },
  //生命周期 - 挂载完成（可以访问 DOM 元素）
  mounted() {},
  beforeCreate() {}, //生命周期 - 创建之前
  beforeMount() {}, //生命周期 - 挂载之前
  beforeUpdate() {}, //生命周期 - 更新之前
  updated() {}, //生命周期 - 更新之后
  beforeDestroy() {}, //生命周期 - 销毁之前
  destroyed() {}, //生命周期 - 销毁完成
  activated() {}, //如果页面有 keep-alive 缓存功能，这个函数会触发
};
</script>
<style  scoped>
</style>