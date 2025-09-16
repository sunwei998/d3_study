  // 一级节点外发光滤镜
  defs.append('filter')
    .attr('id', 'root-glow')
    .html(`
      <feDropShadow dx="0" dy="0" stdDeviation="8" flood-color="#fff" flood-opacity="0.7"/>
    `);
<template>
  <div class="home">
    <svg ref="svgRef" style="display:block;margin:0 32px;width:calc(100vw - 64px);height:600px;"></svg>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue'
import * as d3 from 'd3'

const svgRef = ref(null)

// mock 树形结构数据
const treeData = {
  id: '10001',
  level: 1,
  name: '小王玩机',
  children: [
    {
      id: '20012',
      level: 2,
      name: '阿狸的日常',
      children: [
        { id: '30123', level: 3, name: '小明学编程' },
        { id: '30234', level: 3, name: '大壮健身记' },
        { id: '30345', level: 3, name: '小美的厨房' }
      ]
    },
    {
      id: '20023',
      level: 2,
      name: '小红爱旅游',
      children: [
        { id: '30456', level: 3, name: '老李摄影记' },
        { id: '30567', level: 3, name: '小刚的日记' },
        { id: '30678', level: 3, name: '阿花的宠物屋' },
        { id: '30789', level: 3, name: '小七的手账' },
        { id: '30890', level: 3, name: '大牛数码控' }
      ]
    },
    {
      id: '20034',
      level: 2,
      name: '大熊的生活',
      children: [
        { id: '30901', level: 4, name: '小雨的画室' },
        { id: '31012', level: 4, name: '小宇的乐队' },
        { id: '31123', level: 4, name: '小艾的花园' }
      ]
    }
  ]
}

// 递归转换为 nodes/links
function treeToGraph(tree) {
  const nodes = []
  const links = []
  function traverse(node, parent = null) {
    nodes.push({ id: node.id, level: node.level, name: node.name })
    if (parent) {
      links.push({ source: parent.id, target: node.id })
    }
    if (node.children && node.children.length) {
      node.children.forEach(child => traverse(child, node))
    }
  }
  traverse(tree)
  return { nodes, links }
}

const { nodes, links } = treeToGraph(treeData)

onMounted(() => {
  let width = Math.max(window.innerWidth - 64, 400)
  const height = 600
  const svg = d3.select(svgRef.value)
  svg.selectAll('*').remove()
  svg.attr('viewBox', `0 0 ${width} ${height}`)

  // 定义阴影和高光滤镜
  const defs = svg.append('defs');
  // 多种半径的箭头marker
  const markerSizes = [12, 16, 22, 36];
        markerSizes.forEach(r => {
          defs.append('marker')
            .attr('id', `arrow-${r}`)
            .attr('viewBox', '0 -2.5 5 5')
            .attr('refX', r)
            .attr('refY', 0)
            .attr('markerWidth', 5)
            .attr('markerHeight', 5)
            .attr('orient', 'auto')
            .append('path')
            .attr('d', 'M0,-2.5L5,0L0,2.5')
            .attr('fill', '#666');
        });

  // 阴影滤镜
  defs.append('filter')
    .attr('id', 'shadow')
    .html(`
      <feDropShadow dx="0" dy="4" stdDeviation="6" flood-color="#888" flood-opacity="0.35"/>
    `);
  // 镜面高光滤镜
  defs.append('radialGradient')
    .attr('id', 'highlight')
    .attr('cx', '30%')
    .attr('cy', '30%')
    .attr('r', '70%')
    .selectAll('stop')
    .data([
      { offset: '0%', color: 'rgba(255,255,255,0.7)' },
      { offset: '40%', color: 'rgba(255,255,255,0.2)' },
      { offset: '100%', color: 'rgba(255,255,255,0)' }
    ])
    .enter()
    .append('stop')
    .attr('offset', d => d.offset)
    .attr('stop-color', d => d.color);

  // 力导向仿真
  let simulation = d3.forceSimulation(nodes)
    .force('link', d3.forceLink(links).id(d => d.id).distance(100))
    .force('charge', d3.forceManyBody().strength(-300))
    .force('center', d3.forceCenter(width / 2, height / 2))
    .force('collide', d3.forceCollide(30))

  // 响应窗口变化
  function resize() {
    width = Math.max(window.innerWidth - 64, 400)
    svg.attr('viewBox', `0 0 ${width} ${height}`)
    simulation.force('center', d3.forceCenter(width / 2, height / 2))
    simulation.alpha(1).restart()
  }
  window.addEventListener('resize', resize)

  onBeforeUnmount(() => {
    window.removeEventListener('resize', resize)
    simulation.stop()
  })

  // 族群分组
  // 自动生成 groupMap：每个二级节点（level 2）为一个分组，包含自己和所有子孙节点 id
  function getAllIds(node) {
    let ids = [node.id];
    if (node.children && node.children.length) {
      node.children.forEach(child => {
        ids = ids.concat(getAllIds(child));
      });
    }
    return ids;
  }
  const groupMap = {};
  if (treeData.children && treeData.children.length) {
    treeData.children.forEach(child => {
      groupMap[child.id] = getAllIds(child);
    });
  }
  // 明显区分的浅色生成函数（分段色相）
  function distinctLightColor(idx, total, alpha = 1) {
    // 色相分段，保证区分度
    const h = Math.round((idx * 360) / total);
    const s = 65;
    const l = 85;
    if (alpha === 1) {
      return `hsl(${h}, ${s}%, ${l}%)`;
    } else {
      return `hsla(${h}, ${s}%, ${l}%, ${alpha})`;
    }
  }
  // 一级节点颜色
  const rootColor = distinctLightColor(0, Object.keys(groupMap).length + 1);
  // 族群颜色
  const groupColorMap = {};
  const groupAlphaColorMap = {};
  const groupKeys = Object.keys(groupMap);
  groupKeys.forEach((group, idx) => {
    groupColorMap[group] = distinctLightColor(idx + 1, groupKeys.length + 1, 1);
    groupAlphaColorMap[group] = distinctLightColor(idx + 1, groupKeys.length + 1, 0.38);
  });
  // 节点id到族群的映射
  const nodeGroupMap = {};
  Object.entries(groupMap).forEach(([group, ids]) => {
    ids.forEach(id => {
      nodeGroupMap[id] = group;
    });
  });
  const shadowLayer = svg.append('g')
  function updateShadows() {
    shadowLayer.selectAll('*').remove()
    Object.entries(groupMap).forEach(([group, ids]) => {
      const members = nodes.filter(n => ids.includes(n.id));
      if (members.length) {
        // 以成员节点均值为圆心
        let x = d3.mean(members, d => d.x || 0);
        let y = d3.mean(members, d => d.y || 0);
        // 计算所有成员节点到圆心的最大距离+节点半径+padding
        const getNodeR = d => {
          if (d.level === 1) return 36;
          if (d.level === 2) return 22;
          if (d.level === 3) return 16;
          if (d.level === 4) return 12;
          return 10;
        };
        let maxDist = 0;
        members.forEach(m => {
          const dx = (m.x || 0) - x;
          const dy = (m.y || 0) - y;
          const dist = Math.sqrt(dx*dx + dy*dy) + getNodeR(m);
          if (dist > maxDist) maxDist = dist;
        });
        let r = maxDist + 18; // padding
        // 保证不超出边界
        x = Math.max(r, Math.min(width - r, x));
        y = Math.max(r, Math.min(height - r, y));
        r = Math.min(r, x, width - x, y, height - y);
        // 族群阴影圆底色（更低透明度）
        shadowLayer.append('circle')
          .attr('cx', x)
          .attr('cy', y)
          .attr('r', r)
          .attr('fill', groupAlphaColorMap[group])
          .attr('filter', 'url(#shadow)');
        shadowLayer.append('circle')
          .attr('cx', x)
          .attr('cy', y)
          .attr('r', r)
          .attr('fill', 'url(#highlight)')
          .attr('pointer-events', 'none');
      }
    });
  }

  // 画连线（带箭头）
  // 画连线（根据target节点半径动态选择marker）
  const link = svg.append('g')
  .attr('stroke', '#666')
    .attr('stroke-opacity', 0.8)
    .selectAll('line')
    .data(links)
    .join('line')
  .attr('stroke-width', 1.2)
    .attr('marker-end', d => {
      // 找到target节点半径
      const target = nodes.find(n => n.id === (d.target.id || d.target));
      let r = 16;
      if (target) {
        if (target.level === 1) r = 36;
        else if (target.level === 2) r = 22;
        else if (target.level === 3) r = 16;
        else if (target.level === 4) r = 12;
      }
      return `url(#arrow-${r})`;
    })

  // 画节点
  const node = svg.append('g')
    .selectAll('circle')
    .data(nodes)
    .join('circle')
    .attr('stroke', d => d.level === 1 ? '#222' : '#fff')
    .attr('stroke-width', d => d.level === 1 ? 4 : 1.5)
    .attr('filter', d => d.level === 1 ? 'url(#root-glow)' : null)
    .attr('r', d => {
      if (d.level === 1) return 36 // 一级节点更大
      if (d.level === 2) return 22
      if (d.level === 3) return 16
      if (d.level === 4) return 12
      return 10
    })
    .attr('fill', d => {
      if (d.level === 1) return rootColor;
      const group = nodeGroupMap[d.id];
      if (group) return groupColorMap[group];
      return '#eee';
    })
    // 一级节点加白色高光圈
    .each(function(d) {
      if (d.level === 1) {
        d3.select(this.parentNode)
          .insert('circle', ':first-child')
          .attr('cx', d.x)
          .attr('cy', d.y)
          .attr('r', 44)
          .attr('fill', 'none')
          .attr('stroke', '#fff')
          .attr('stroke-width', 8)
          .attr('opacity', 0.7);
      }
    })
    .style('cursor', 'pointer')
    .on('click', (event, d) => {
      console.log('节点信息:', d)
    })

  // 节点文字
  const label = svg.append('g')
    .selectAll('text')
    .data(nodes)
    .join('text')
  .attr('text-anchor', 'middle')
  .attr('dy', 5)
  .style('font-weight', 'bold')
  .attr('fill', '#222')
  .text(d => d.name)

  simulation.on('tick', () => {
    updateShadows()
    // 限制节点在视野内
    nodes.forEach(d => {
      d.x = Math.max(20, Math.min(width - 20, d.x))
      d.y = Math.max(20, Math.min(height - 20, d.y))
    })
    link
      .attr('x1', d => d.source.x)
      .attr('y1', d => d.source.y)
      .attr('x2', d => d.target.x)
      .attr('y2', d => d.target.y)
    node
      .attr('cx', d => d.x)
      .attr('cy', d => d.y)
    label
      .attr('x', d => d.x)
      .attr('y', d => d.y)
  })
})
</script>