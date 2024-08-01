<template>
  <div>
    <textarea v-model="inputValue" :placeholder="placeHolder" class="fixed-input"></textarea>
    <button @click="drawNetwork" class="fixed-button">Generate Diagram</button>
    <div ref="visNetwork" class="vis-network"></div>
    <button @click="goToGithub" class="github-button">Github Project</button>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue';
import { Network } from 'vis-network/standalone'; // 引入 standalone 版本

const goToGithub = () => {
  window.location.href = 'https://github.com/bittervan/TraceScope';
}

const inputValue = ref(` 5)               |  lru_add_fn() {
 5)   0.090 us    |    __rcu_read_lock();
 5)   0.090 us    |    folio_mapping();
 5)   0.090 us    |    __rcu_read_unlock();
 5)               |    __mod_lruvec_state() {
 5)   0.090 us    |      __mod_node_page_state();
 5)               |      __mod_memcg_lruvec_state() {
 5)   0.100 us    |        cgroup_rstat_updated();
 5)   0.280 us    |      }
 5)   0.650 us    |    }
 5)   0.100 us    |    __mod_zone_page_state();
 5)   0.100 us    |    mem_cgroup_update_lru_size();
 5)   1.800 us    |  }
`);
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

class MigrationTrace {
  constructor() {
    this.logs = [];
    this.elapses = [];
  }
}

function lines_to_times(lines) {
  const times = [];
  const pattern = /(\d+(\.\d+)?)/;
  lines.forEach(line => {
    const match = line.substring(4).match(pattern);
    if (match) {
      const time = parseFloat(match[1]);
      times.push(time);
    } else {
      times.push(-1);
    }
  });
  return times;
}

function lines_to_trace(lines) {
  const trace = new MigrationTrace();
  trace.logs = lines.map(line => line.trim().substring(20).trim());
  trace.elapses = lines_to_times(lines);
  return trace;
}

class TreeNode {
  constructor(value) {
    this.value = value;
    this.children = [];
  }
}

function parse_logs_to_tree(logs) {
  function parse_node(index) {
    const node_value = logs[index];
    const node = new TreeNode(node_value);
    index++;
    while (index < logs.length && logs[index].trim() !== '}') {
      if (logs[index].includes('{')) {
        const [child, newIndex] = parse_node(index);
        node.children.push(child);
        index = newIndex;
      } else {
        node.children.push(new TreeNode(logs[index]));
        index++;
      }
    }
    return [node, index + 1];
  }

  const [root] = parse_node(0);
  return root;
}

function trim_tree_node(node) {
  node.value = node.value.trim();
  if (node.value.endsWith('{') || node.value.endsWith(';') || node.value.endsWith('()')) {
    node.value = node.value.slice(0, -1).trim();
  }
  if (node.value.includes('}')) {
    return null;
  }
  node.children = node.children.filter(child => {
    const trimmed_child = trim_tree_node(child);
    return trimmed_child !== null;
  });
  return node;
}

function accumulate_time(tree, trace) {
  function create_time_tree(node) {
    const time_node = {
      time: 0,
      children: node.children.map(create_time_tree)
    };
    return time_node;
  }

  const time_tree = create_time_tree(tree);

  function accumulate_time_recursive(node, times, index) {
    if (times[index] === -1) {
      node.children.forEach(child => {
        index = accumulate_time_recursive(child, times, index + 1);
      });
      index++;
    }
    node.time += times[index];
    return index;
  }

  accumulate_time_recursive(time_tree, trace.elapses, 0);
  return time_tree;
}

function hsv_to_rgb(h, s, v) {
  let r, g, b;
  const i = Math.floor(h * 6);
  const f = h * 6 - i;
  const p = v * (1 - s);
  const q = v * (1 - f * s);
  const t = v * (1 - (1 - f) * s);
  switch (i % 6) {
    case 0: r = v; g = t; b = p; break;
    case 1: r = q; g = v; b = p; break;
    case 2: r = p; g = v; b = t; break;
    case 3: r = p; g = q; b = v; break;
    case 4: r = t; g = p; b = v; break;
    case 5: r = v; g = p; b = q; break;
  }
  return [Math.round(r * 255), Math.round(g * 255), Math.round(b * 255)];
}

function get_color(time, parent_time, max_time) {
  if (time == max_time) {
    return `rgba(255, 0, 0)`
  }
  // console.log(time, parent_time, max_time)
  const hue = 210 - 210 * time / max_time;
  const saturation = 90 * time / parent_time + 10;
  const [r, g, b] = hsv_to_rgb(hue / 360, saturation / 100, 1);
  return `rgba(${r}, ${g}, ${b})`;
}

function generate_vis_dict(func_tree, time_tree, max_time) {
  const nodes = [];
  const edges = [];

  function walk_tree(node, time_node, index, depth, parent_time) {
    const color = get_color(time_node.time, parent_time, max_time);
    nodes.push({
      label: `${node.value}\n${(time_node.time * 100 / max_time).toFixed(2)}%`,
      color: { background: color, border: 'black' },
      id: index,
      level: depth
    });
    let currentIndex = index;
    node.children.forEach((child, i) => {
      edges.push({ from: currentIndex, to: index + 1 });
      index = walk_tree(child, time_node.children[i], index + 1, depth + 1, time_node.time);
    });
    return index;
  }

  walk_tree(func_tree, time_tree, 0, 0, max_time);

  const nodesDict = nodes.reduce((acc, node) => {
    acc[node.id] = node;
    return acc;
  }, {});

  const edgesDict = edges.reduce((acc, edge, i) => {
    acc[i] = edge;
    return acc;
  }, {});

  return { nodes: nodesDict, edges: edgesDict };
}

function log_to_vis(raw_log) {
  const log = raw_log.trim();
  const log_lines = log.split("\n");
  const trace = lines_to_trace(log_lines);
  const tree = parse_logs_to_tree(trace.logs);
  const trimmedTree = trim_tree_node(tree);
  const timeTree = accumulate_time(trimmedTree, trace);
  const visDict = generate_vis_dict(trimmedTree, timeTree, Math.max(...trace.elapses));
  return visDict;
}

const drawNetwork = () => {

  const tree_data = log_to_vis(inputValue.value)
  // console.log(tree_data)
  // 节点和边数据
  // const nodes = [
  //   { id: 1, label: 'Node 1' },
  //   { id: 2, label: 'Node 2' },
  //   { id: 3, label: 'Node 3' },
  //   { id: 4, label: 'Node 4' },
  //   { id: 5, label: 'Node 5' }
  // ];
  const nodes = Object.values(tree_data.nodes)
  // console.log(nodes)

  // const edges = [
  //   { from: 1, to: 2 },
  //   { from: 1, to: 3 },
  //   { from: 2, to: 4 },
  //   { from: 2, to: 5 }
  // ];
  const edges = Object.values(tree_data.edges)
  // console.log(edges)

  // const container = this.$refs.visNetwork;
  const container = document.querySelector('.vis-network');
  const data = { nodes, edges };
  const options = {
    edges: {
      smooth: {
        type: "cubicBezier",
        forceDirection: "horizontal",
        roundness: 0.4,
      },
    },
    layout: {
      hierarchical: {
        direction: "LR"
      },
    },
    physics: {
        stabilization: true,
        barnesHut: {
            gravitationalConstant: -30000,
            centralGravity: 0.3,
            springLength: 100,
            springConstant: 0.04,
            damping: 0.09,
            avoidOverlap: 0.2
        }
    },
  };

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
  width: calc(25vw);
}

.fixed-button {
  position: fixed;
  top: 10px;
  left: calc(12.5vw + 20px - 5px);
  height: 30px;
  width: calc(12.5vw - 5px);
}

.github-button {
  position: fixed;
  top: 10px;
  left: 10px;
  height: 30px;
  width: calc(12.5vw - 5px);
}

.vis-network {
  width: calc(80vw - 30px); /* 设置图框的宽度 */
  height: calc(100vh - 20px); /* 设置图框的高度 */
  border: 1px solid gray;
  margin-top: 0px;
  position: fixed;
  top: 10px;
  left: calc(25vw + 20px);
}
</style>