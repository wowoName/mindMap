<template>
  <div class="mind-map-container">
    <!-- <el-button type="primary" size="small">自适应</el-button> -->
    <mind-map class="mind-nap-main" ref="mindMap" @selected="selected" @addNode="addNode" @updateNode="updateNode">
    </mind-map>

    <el-dialog title="新增节点" :visible.sync="dialogFormVisible">
      <el-form :model="form" :rules="rules" ref="ruleForm" label-width="120px">
        <el-form-item label="节点名称：" prop="name">
          <el-input v-model="form.name" autocomplete="off" placeholder="请输入节点名称"></el-input>
        </el-form-item>
        <el-form-item label="节点背景色：">
          <el-color-picker v-model="form.color"></el-color-picker>
        </el-form-item>
        <el-form-item label="节点位置：" v-if="isTreeRoot">
          <el-radio-group v-model="form.dir" size="small">
            <el-radio-button label="left">左侧</el-radio-button>
            <el-radio-button label="right">右侧</el-radio-button>
          </el-radio-group>
        </el-form-item>
        <!--  <el-form-item label="宽度：" >
          <el-input v-model="form.w" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="高度：" >
          <el-input v-model="form.h" autocomplete="off"></el-input>
        </el-form-item> -->
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button @click="dialogFormVisible = false">取 消</el-button>
        <el-button type="primary" @click="handlerAddNode()">确 定</el-button>
      </div>
    </el-dialog>
  </div>
</template>

<script>
import MindMap from './Index.vue'
export default {
  data() {
    return {
      dialogFormVisible: false,
      isAdd: false,// 是否新增节点标识
      isTreeRoot: false,// 是否为根节点
      form: {
        name: '',
        color: '#397bf7',
        w: 120,
        h: 40,
        key: '',
        dir: "right",// 节点方向 默认为右侧
      },
      rules: {
        name: [
          { required: true, message: '请输入名称', trigger: 'blur' }
        ]
      }
    }
  },
  components: {
    MindMap
  },
  methods: {
    /**
     * 添加、编辑节点
     */
    handlerAddNode() {
      this.$refs.ruleForm.validate((valid) => {
        if (valid) {
          const nodeData = Object.assign({}, this.form)
          // 新增节点
          if (this.isAdd) {
            nodeData.key = (Math.random() * (36 - 1) + 1).toString(32).substr(3, 6)
            this.$refs.mindMap.addMindmapNodeData(nodeData);
          } else {// 更新节点
            this.$refs.mindMap.updateNodeData(nodeData);
          }
          this.dialogFormVisible = false
        } else {
          console.log('error submit!!');
          return false;
        }
      });

    },
    // 节点选择change事件
    selected(item) {
      console.log('选择了事件', item)
    },
    /**
     * 右键添加节点
     * @param {*} isTreeRoot 当前点击节点是否为根节点
     */
    addNode({ isTreeRoot }) {
      this.isAdd = true
      this.dialogFormVisible = true
      this.isTreeRoot = isTreeRoot
      this.form = Object.assign({}, this.$options.data().form)
      this.$nextTick(() => {
        this.$refs.ruleForm.resetFields();
      })
    },
    /**
     * 编辑节点
     * @param {*} nodeData  节点数据
     */
    updateNode({ nodeData, isTreeRoot }) {
      this.dialogFormVisible = true
      this.isAdd = false
      this.isTreeRoot = isTreeRoot
      this.$nextTick(() => {
        this.$refs.ruleForm.resetFields();
        this.form = nodeData
      })
    }
  }
}
</script>

<style lang='less' scoped>
.mind-map-container {
  height: 100%;
  width: 100%;
  display: flex;
  align-items: flex-start;
  flex-direction: column;
  .mind-nap-main {
    flex: 1;
  }
}
</style>