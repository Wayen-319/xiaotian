<template>
  <div class="container">
    <h2 style="margin-bottom: 20px;">树状图示例</h2>
    <div 
      id="graph-chart-container" 
      style="
        position: relative;
        width: min(90vw, 1200px);
        height: min(70vh, 800px);
        border: 1px solid #eee;
        background: #fafbfc;
        /* overflow: auto;  移除滚动条 */
        box-sizing: border-box;
      "
    ></div>
    <div v-if="error" class="error">{{ error }}</div>
  </div>
</template>

<style>
.tooltip {  
  position: absolute;
  min-width: 400px;
  max-width: 800px;
  padding: 15px;
  font-size: 14px;
  text-align: left;
  color: black;
  border: 2px solid #ccc;
  background-color: #fff;
  border-radius: 8px;
  box-shadow: 0 0 15px rgba(0,0,0,0.2);
  pointer-events: auto;
  z-index: 10;
  line-height: 1.5;
  white-space: normal;
  word-wrap: break-word;
  overflow-wrap: break-word;
  font-family: Arial, sans-serif;
}

.tooltip.persistent {
  border-color: #219ebc;
}

.tooltip .close-btn {
  position: absolute;
  top: 5px;
  right: 5px;
  width: 20px;
  height: 20px;
  line-height: 20px;
  text-align: center;
  cursor: pointer;
  color: #666;
  font-size: 16px;
  border-radius: 50%;
  background: #f0f0f0;
  display: none;  /* 默认隐藏关闭按钮 */
}

.tooltip.persistent .close-btn {
  display: block;  /* 常驻状态显示关闭按钮 */
}

.tooltip .close-btn:hover {
  background: #e0e0e0;
  color: #333;
}

.tooltip .content {
  margin-right: 20px;
  max-height: 400px;
  overflow-y: auto;
  overflow-x: auto;
  white-space: normal;
}

.tooltip .content pre {
  margin: 0;
  padding: 0;
  white-space: pre-wrap;
  word-wrap: break-word;
  overflow-x: auto;
  overflow-y: auto;
  max-height: 300px;
  font-family: Arial, sans-serif;
  display: block;
  width: 100%;
}

.error {
  color: red;
  margin-top: 20px;
}
</style>

<script setup>
import { onMounted, ref } from 'vue'
import * as d3 from 'd3'

const error = ref(null)

// 添加HTML转义函数
function escapeHtml(text) {
  const div = document.createElement('div');
  div.textContent = text;
  return div.innerHTML;
}

// 画树状图（竖直方向，根节点在最上边，支持缩放）
function drawTreeChart(treeData, data) {
  d3.select("#graph-chart-container").selectAll("*").remove();

  // 画布尺寸大一些
  const width = 3200;
  const height = 2400;
  const nodeWidth = 120;
  const nodeHeight = 60;
  const imgSize = 40;

  const svg = d3.select("#graph-chart-container")
    .append("svg")
    .attr("width", width)
    .attr("height", height)
    .style("display", "block");

  // 添加缩放行为
  const zoom = d3.zoom()
    .scaleExtent([0.05, 10])
    .on("zoom", (event) => {
      g.attr("transform", event.transform);
    });

  svg.call(zoom);

  // 创建层次结构
  const root = d3.hierarchy(treeData);

  // 创建树布局（竖直方向，根节点在最上边）
  const treeLayout = d3.tree()
    .size([width - 200, height - 120]) // [x, y]，x横向，y纵向
    .nodeSize([10, 600]);

  treeLayout(root);

  // 创建一个g用于缩放
  const g = svg.append("g");

  // 画连线
  g.append("g")
    .selectAll("path")
    .data(root.links().map(link => {
      const originalLink = data.links.find(l => 
        l.source === link.source.data.name && 
        l.target === link.target.data.name
      );
      return {
        ...link,
        content: originalLink?.content || "No content available"
      };
    }))
    .join("path")
    .attr("fill", "none")
    .attr("stroke", "#999")
    .attr("stroke-width", 2)
    .attr("d", d3.linkVertical()
      .source(d => [d.source.x+100, d.source.y+60])
      .target(d => [d.target.x+100, d.target.y+60])
    )
    .on("mouseover", (event, d) => {
      if (!tooltip.classed("persistent")) {
        d3.select(event.currentTarget)
          .attr("stroke", "#219ebc")
          .attr("stroke-width", 4);
        tooltip.transition().duration(200).style("opacity", 1);
        const linkContent = d.content || "No content available";
        tooltip.select(".content")
          .html(`<pre>${escapeHtml(linkContent)}</pre>`)
          .style("left", (event.pageX + 20) + "px")
          .style("top", (event.pageY - 20) + "px");
      }
    })
    .on("mousemove", (event) => {
      if (!tooltip.classed("persistent")) {
        tooltip.style("left", (event.pageX + 20) + "px")
              .style("top", (event.pageY - 20) + "px");
      }
    })
    .on("mouseout", () => {
      if (!tooltip.classed("persistent")) {
        d3.select(event.currentTarget)
          .attr("stroke", "#999")
          .attr("stroke-width", 2);
        tooltip.transition().duration(200).style("opacity", 0);
      }
    })
    .on("click", (event, d) => {
      event.stopPropagation();
      const isPersistent = tooltip.classed("persistent");
      tooltip.classed("persistent", !isPersistent);
      
      if (!isPersistent) {
        tooltip.transition().duration(200).style("opacity", 1);
        const linkContent = d.content || "No content available";
        tooltip.select(".content")
          .html(`<pre>${escapeHtml(linkContent)}</pre>`)
          .style("left", (event.pageX + 20) + "px")
          .style("top", (event.pageY - 20) + "px");
      } else {
        tooltip.transition().duration(200).style("opacity", 0);
      }
    });

  // tooltip
  const tooltip = d3.select("body")
    .append("div")
    .attr("class", "tooltip")
    .style("opacity", 0)
    .html('<div class="close-btn">×</div><div class="content"></div>');

  // 添加关闭按钮事件
  tooltip.select(".close-btn")
    .on("click", () => {
      tooltip.transition().duration(200).style("opacity", 0);
      tooltip.classed("persistent", false);  // 移除常驻状态
    });

  // 画节点
  const node = g.append("g")
    .selectAll("g")
    .data(root.descendants())
    .join("g")
    .attr("transform", d => `translate(${d.x + 100},${d.y + 60})`);

  // 节点
  node.append("circle")
    .attr("r", 2)
    .attr("fill", "#8ecae6")
    .attr("stroke", "#219ebc")
    .attr("stroke-width", 3);

  // 节点文字，自动居中
  node.append("text")
    .attr("y", 7)
    .attr("font-size", 2)
    .attr("fill", "#222")
    .attr("text-anchor", "middle") // 居中对齐
    .text(d => d.data.name);

  // 修改节点 tooltip 事件
  node.on("mouseover", (event, d) => {
    if (!tooltip.classed("persistent")) {
      tooltip.transition().duration(200).style("opacity", 1);
      tooltip.select(".content")
        .html(`<pre>${escapeHtml(d.data.intro)}</pre>`)
        .style("left", (event.pageX + 20) + "px")
        .style("top", (event.pageY - 20) + "px");
    }
  })
  .on("mousemove", (event) => {
    if (!tooltip.classed("persistent")) {
      tooltip.style("left", (event.pageX + 20) + "px")
             .style("top", (event.pageY - 20) + "px");
    }
  })
  .on("mouseout", () => {
    if (!tooltip.classed("persistent")) {
      tooltip.transition().duration(200).style("opacity", 0);
    }
  })
  .on("click", (event, d) => {
    event.stopPropagation();
    const isPersistent = tooltip.classed("persistent");
    tooltip.classed("persistent", !isPersistent);
    
    if (!isPersistent) {
      tooltip.transition().duration(200).style("opacity", 1);
      tooltip.select(".content")
        .html(`<pre>${escapeHtml(d.data.intro)}</pre>`)
        .style("left", (event.pageX + 20) + "px")
        .style("top", (event.pageY - 20) + "px");
    } else {
      tooltip.transition().duration(200).style("opacity", 0);
    }
  });
}

onMounted(async () => {
  try {
    const res = await fetch('xiaotian/b.json');
    if (!res.ok) throw new Error('b.json not found, using mock data');
    const data = await res.json();
    
    // 创建节点映射
    const idMap = {};
    data.nodes.forEach(n => idMap[n.name] = { ...n, children: [] });
    
    // 创建link映射，用于存储每个link的content
    const linkMap = {};
    data.links.forEach(l => {
      linkMap[`${l.source}-${l.target}`] = l.content;
    });
    
    // 构建树结构
    data.links.forEach(l => {
      if (idMap[l.source] && idMap[l.target]) {
        idMap[l.source].children = idMap[l.source].children || [];
        // 把link的content也加入到子节点数据中
        const childNode = { ...idMap[l.target], linkContent: linkMap[`${l.source}-${l.target}`] };
        idMap[l.source].children.push(childNode);
      }
    });
    
    // 找到根节点
    const targets = new Set(data.links.map(l => l.target));
    const rootNode = data.nodes.find(n => !targets.has(n.name)) || data.nodes[0];
    drawTreeChart(idMap[rootNode.name], data);  // 传入 data
  } catch (e) {
    error.value = "onMounted行发生错误";
    console.log(e);
  }
});
</script>

<style scoped>
.container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 30px 0;
}
#graph-chart-container {
  /* 移除滚动条，支持缩放 */
  overflow: hidden;
}
</style>
