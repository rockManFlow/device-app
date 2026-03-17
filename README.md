
![项目结构介绍](README_files/1.jpg)

## vue文件介绍
1. 一个 .vue 文件包含 3 个核心部分
* <template>
  <!-- 1. 模板区：页面/组件的结构（必须有且只能有一个根节点） -->
</template>

<script>
// 2. 逻辑区：数据、方法、生命周期、交互逻辑（Vue 核心）
export default {
  // 页面/组件名称
  name: "Index",
  // 响应式数据
  data() {
    return {
      msg: "Hello UniApp + Vue"
    };
  },
  // 生命周期钩子（UniApp 扩展了小程序/APP 特有的钩子）
  onLoad(options) {
    console.log("页面加载", options);
  },
  // 方法
  methods: {
    handleClick() {
      this.msg = "点击后修改";
    }
  }
};
</script>

<style scoped>
/* 3. 样式区：页面/组件样式（scoped 表示样式仅作用于当前组件） */
.content {
}
</style>