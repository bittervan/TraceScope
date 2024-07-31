<template>
  <div>
    <textarea v-model="inputValue" :placeholder="placeHolder" class="fixed-input"></textarea>
    <button @click="drawNetwork" class="fixed-button">Generate Diagram</button>
    <div ref="visNetwork" class="vis-network"></div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue';
import { Network } from 'vis-network/standalone'; // 引入 standalone 版本

const inputValue = ref('');
const placeHolder = ref(`cd /sys/kernel/debug/tracing
echo 0 > tracing_on
echo > trace
echo <function> > set_graph_function
echo function_graph > current_tracer
echo 1 > tracing_on
cat trace

And paste one function tree here.
Then click Generate Diagram above.
`);

const drawNetwork = () => {
  // 节点和边数据
  const nodes = [
    { id: 1, label: 'Node 1' },
    { id: 2, label: 'Node 2' },
    { id: 3, label: 'Node 3' },
    { id: 4, label: 'Node 4' },
    { id: 5, label: 'Node 5' }
  ];

  const edges = [
    { from: 1, to: 2 },
    { from: 1, to: 3 },
    { from: 2, to: 4 },
    { from: 2, to: 5 }
  ];

  // const container = this.$refs.visNetwork;
  const container = document.querySelector('.vis-network');
  const data = { nodes, edges };
  const options = {};

  new Network(container, data, options);
};

onMounted(() => {
  drawNetwork();
});
</script>

<style scoped>
.fixed-input {
  position: fixed;
  top: 50px;
  left: 10px;
  height: calc(100vh - 60px);
  width: calc(20vw);
}

.fixed-button {
  position: fixed;
  top: 10px;
  left: 10px;
  height: 30px;
  width: calc(20vw);
}

.vis-network {
  width: calc(80vw - 30px); /* 设置图框的宽度 */
  height: calc(100vh - 20px); /* 设置图框的高度 */
  border: 1px solid gray;
  margin-top: 0px;
  position: fixed;
  top: 10px;
  left: calc(20vw + 20px);
}
</style>