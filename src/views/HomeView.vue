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
  // 一级节点外发光滤镜
  defs.append('filter')
    .attr('id', 'root-glow')
    .attr('x', '-40%')
    .attr('y', '-40%')
    .attr('width', '180%')
    .attr('height', '180%')
    .html(`
      <feDropShadow dx="0" dy="0" stdDeviation="8" flood-color="#fff" flood-opacity="0.7"/>
    `);
  // 箭头marker，refX 设为箭头宽度（5），让箭头尖端正好落在连线末端
  defs.append('marker')
    .attr('id', 'arrow-circle')
    .attr('viewBox', '0 -2.5 5 5')
    .attr('refX', 5)
    .attr('refY', 0)
    .attr('markerWidth', 5)
    .attr('markerHeight', 5)
    .attr('orient', 'auto')
    .append('path')
    .attr('d', 'M0,-2.5L5,0L0,2.5')
    .attr('fill', '#666');
  defs.append('marker')
    .attr('id', 'arrow-rect')
    .attr('viewBox', '0 -2.5 5 5')
    .attr('refX', 5)
    .attr('refY', 0)
    .attr('markerWidth', 5)
    .attr('markerHeight', 5)
    .attr('orient', 'auto')
    .append('path')
    .attr('d', 'M0,-2.5L5,0L0,2.5')
    .attr('fill', '#666');

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
  // 动态计算节点半径，防止重叠
  function getNodeRadius(d) {
    if (!d || !d._textW) return 30;
    if (d.level === 1) return Math.max(36, d._textW/2 + 14);
    return Math.max((d._textW + 32) / 2, 38 / 2); // 矩形节点取宽高最大值
  }
  // 动态计算连线距离，更紧凑的布局
  function getLinkDistance(d) {
    const s = d.source, t = d.target;
    // 一级节点和二级节点之间的距离（考虑更大的一级节点）
    if (s.level === 1 && t.level === 2) return 120;
    // 二级节点和三级节点之间的距离（由于文字变小，距离可以更近）
    if ((s.level === 2 && t.level === 3) || (s.level === 3 && t.level === 2)) return 60;
    // 其他连接
    return 50 + Math.max(getNodeRadius(s), getNodeRadius(t));
  }

  // 获取节点或组的边界信息
  function getBoundaryInfo(nodes, group = null) {
    const filteredNodes = group ? nodes.filter(n => nodeGroupMap[n.id] === group) : nodes;
    if (!filteredNodes.length) return null;

    let minX = Infinity, minY = Infinity;
    let maxX = -Infinity, maxY = -Infinity;
    
    filteredNodes.forEach(n => {
      const radius = getNodeRadius(n);
      minX = Math.min(minX, n.x - radius);
      minY = Math.min(minY, n.y - radius);
      maxX = Math.max(maxX, n.x + radius);
      maxY = Math.max(maxY, n.y + radius);
    });

    const width = maxX - minX;
    const height = maxY - minY;
    const centerX = minX + width / 2;
    const centerY = minY + height / 2;
    
    return { minX, minY, maxX, maxY, width, height, centerX, centerY };
  }

  // 创建边界感知和群组内吸引力
  function adaptiveForce(alpha) {
    const padding = 40; // 边界padding
    const bounds = { minX: padding, minY: padding, maxX: width - padding, maxY: height - padding };
    
    // 计算整体布局的当前边界
    const globalBounds = getBoundaryInfo(nodes);
    if (!globalBounds) return;

    // 计算需要的缩放比例
    const scaleX = (bounds.maxX - bounds.minX) / globalBounds.width;
    const scaleY = (bounds.maxY - bounds.minY) / globalBounds.height;
    const scale = Math.min(scaleX, scaleY);
    
    // 对每个节点应用力
    for (const node of nodes) {
      // 1. 边界限制力
      const nodeRadius = getNodeRadius(node);
      if (node.x - nodeRadius < bounds.minX) {
        node.x += (bounds.minX - (node.x - nodeRadius)) * alpha;
      }
      if (node.x + nodeRadius > bounds.maxX) {
        node.x += (bounds.maxX - (node.x + nodeRadius)) * alpha;
      }
      if (node.y - nodeRadius < bounds.minY) {
        node.y += (bounds.minY - (node.y - nodeRadius)) * alpha;
      }
      if (node.y + nodeRadius > bounds.maxY) {
        node.y += (bounds.maxY - (node.y + nodeRadius)) * alpha;
      }

      // 2. 群组内吸引力
      if (node.level !== 1) {
        const group = nodeGroupMap[node.id];
        if (group) {
          const groupNodes = nodes.filter(n => nodeGroupMap[n.id] === group);
          const groupBounds = getBoundaryInfo(groupNodes);
          if (groupBounds) {
            // 向群组中心的吸引力
            const dx = (groupBounds.centerX - node.x) * alpha * 0.3;
            const dy = (groupBounds.centerY - node.y) * alpha * 0.3;
            node.x += dx;
            node.y += dy;
            
            // 如果群组超出边界，施加向中心的力
            if (groupBounds.minX < bounds.minX || groupBounds.maxX > bounds.maxX ||
                groupBounds.minY < bounds.minY || groupBounds.maxY > bounds.maxY) {
              const centerForceX = (width/2 - groupBounds.centerX) * alpha * 0.1;
              const centerForceY = (height/2 - groupBounds.centerY) * alpha * 0.1;
              node.x += centerForceX;
              node.y += centerForceY;
            }
          }
        }
      }

      // 3. 全局缩放力（如果整体布局太大）
      if (scale < 1) {
        const dx = (node.x - width/2) * (1 - scale) * alpha;
        const dy = (node.y - height/2) * (1 - scale) * alpha;
        node.x -= dx;
        node.y -= dy;
      }
    }
  }

  let simulation = d3.forceSimulation(nodes)
    .force('link', d3.forceLink(links).id(d => d.id).distance(getLinkDistance))
    .force('charge', d3.forceManyBody()
      .strength(d => {
        // 根据层级和节点数动态调整斥力
        const baseStrength = d.level === 1 ? -400 : -150;
        const neighborCount = links.filter(l => 
          l.source.id === d.id || l.target.id === d.id
        ).length;
        return baseStrength * (1 + neighborCount * 0.1);
      })
    )
    .force('center', d3.forceCenter(width / 2, height / 2))
    .force('collide', d3.forceCollide().radius(d => getNodeRadius(d) + 5))
    .force('adaptive', adaptiveForce)

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
    // 先为每个族群计算实际边界框
    const groupBounds = {};
    Object.entries(groupMap).forEach(([group, ids]) => {
      const members = nodes.filter(n => ids.includes(n.id));
      if (members.length) {
        let minX = Infinity, minY = Infinity;
        let maxX = -Infinity, maxY = -Infinity;
        
        members.forEach(m => {
          if (!m || typeof m.x !== 'number' || typeof m.y !== 'number') return;
          // 获取节点的实际尺寸
          const w = m.level === 1 
            ? Math.max(72, m._textW + 28)  // 圆形节点直径
            : (m._textW + 32);             // 矩形节点宽度
          const h = m.level === 1 ? w : 38; // 矩形高度固定

          // 更新边界框，考虑节点本身的尺寸
          minX = Math.min(minX, m.x - w/2);
          minY = Math.min(minY, m.y - h/2);
          maxX = Math.max(maxX, m.x + w/2);
          maxY = Math.max(maxY, m.y + h/2);
        });

        if (minX !== Infinity) {
          groupBounds[group] = { minX, minY, maxX, maxY };
        }
      }
    });

    // 然后按照边界框绘制族群阴影
    Object.entries(groupBounds).forEach(([group, bounds]) => {
      const { minX, minY, maxX, maxY } = bounds;
      const cx = (minX + maxX) / 2;
      const cy = (minY + maxY) / 2;
      
      // 计算包围圆半径：从中心到最远的边界点
      let r = Math.sqrt(
        Math.max(
          Math.pow(maxX - cx, 2) + Math.pow(maxY - cy, 2),
          Math.pow(minX - cx, 2) + Math.pow(maxY - cy, 2),
          Math.pow(maxX - cx, 2) + Math.pow(minY - cy, 2),
          Math.pow(minX - cx, 2) + Math.pow(minY - cy, 2)
        )
      );

      // 添加适量padding，但不要太大
      r += 8;  // 由于节点更小，padding也相应减少

      // 族群阴影圆底色
      shadowLayer.append('circle')
        .attr('cx', cx)
        .attr('cy', cy)
        .attr('r', r)
        .attr('fill', groupAlphaColorMap[group])
        .attr('filter', 'url(#shadow)');

      // 高光效果
      shadowLayer.append('circle')
        .attr('cx', cx)
        .attr('cy', cy)
        .attr('r', r)
        .attr('fill', 'url(#highlight)')
        .attr('pointer-events', 'none');
    });
  }

  // 画连线（带箭头）
  // 画连线（根据target节点半径动态选择marker）
  // 画连线（适配圆/矩形节点，保证连线有长度且箭头可见）
  const link = svg.append('g')
    .attr('stroke', '#666')
    .attr('stroke-opacity', 0.8)
    .selectAll('line')
    .data(links)
    .join('line')
    .attr('stroke-width', 1.2)
    .attr('marker-end', d => {
      const target = nodes.find(n => n.id === (d.target.id || d.target));
      if (!target) return 'url(#arrow-rect)';
      if (target.level === 1) return 'url(#arrow-circle)';
      return 'url(#arrow-rect)';
    });

  // 文字处理函数：截断过长文本并添加省略号
  function truncateText(text, maxLength = 5) {
    if (text.length <= maxLength) return text;
    return text.slice(0, maxLength) + '...';
  }

  // 获取节点字体大小
  function getFontSize(level) {
    switch(level) {
      case 1: return 20;  // 一级节点
      case 2: return 12;  // 二级节点
      case 3: return 10;  // 三级节点
      default: return 10; // 其他级别
    }
  }

  // 先测量所有节点文本宽度，存到 d._textW 和 d._displayName
  nodes.forEach(d => {
    // 为非一级节点截断文本
    d._displayName = d.level === 1 ? d.name : truncateText(d.name);
    const fontSize = getFontSize(d.level);
    const temp = d3.select('body').append('svg')
      .style('visibility','hidden')
      .append('text')
      .style('font-weight', 'bold')
      .style('font-size', fontSize + 'px')
      .text(d._displayName);
    d._textW = temp.node().getComputedTextLength();
    temp.remove();
  });

  // 创建tooltip div
  const tooltip = d3.select('body').append('div')
    .attr('class', 'node-tooltip')
    .style('position', 'absolute')
    .style('visibility', 'hidden')
    .style('background', '#333')
    .style('color', '#fff')
    .style('padding', '5px 8px')
    .style('border-radius', '4px')
    .style('font-size', '14px')
    .style('pointer-events', 'none')
    .style('z-index', '1000');

  // 一级节点为圆，其他节点为矩形
  const nodeGroup = svg.append('g');
  // 圆形节点
  nodeGroup.selectAll('circle')
    .data(nodes.filter(d => d.level === 1), d => d.id)
    .join('circle')
    .attr('stroke', 'none')
    .attr('stroke-width', 0)
    .attr('filter', 'url(#root-glow)')
    .attr('r', d => Math.max(36, d._textW/2 + 14))
    .attr('fill', rootColor)
    .each(function(d) {
      // 一级节点高光圈
      d3.select(this.parentNode)
        .insert('circle', ':first-child')
        .attr('cx', d.x)
        .attr('cy', d.y)
        .attr('r', Math.max(44, d._textW/2 + 22))
        .attr('fill', 'none')
        .attr('stroke', '#fff')
        .attr('stroke-width', 8)
        .attr('opacity', 0.7);
    })
    .style('cursor', 'pointer')
    .on('click', (event, d) => {
      console.log('节点信息:', d)
    });
  // 矩形节点（非一级）
  nodeGroup.selectAll('rect')
    .data(nodes.filter(d => d.level !== 1), d => d.id)
    .join('rect')
    .attr('stroke', '#fff')
    .attr('stroke-width', 1.5)
    .attr('rx', 16)
    .attr('ry', 16)
    .attr('width', d => d._textW + 32)
    .attr('height', 38)
    .attr('x', d => -(d._textW + 32)/2)
    .attr('y', -19)
    .attr('fill', d => {
      const group = nodeGroupMap[d.id];
      if (group) return groupColorMap[group];
      return '#eee';
    })
    .style('cursor', 'pointer')
    .on('click', (event, d) => {
      console.log('节点信息:', d)
    });

  // 节点文字
  const label = svg.append('g')
    .selectAll('text')
    .data(nodes)
    .join('text')
    .attr('text-anchor', 'middle')
    .attr('dy', d => d.level === 1 ? 6 : 4)
    .style('font-weight', 'bold')
    .style('font-size', d => getFontSize(d.level) + 'px')
    .attr('fill', '#222')
    .text(d => d._displayName)
    .on('mouseover', function(event, d) {
      if (d.name !== d._displayName) {
        tooltip.style('visibility', 'visible')
          .text(d.name);
      }
    })
    .on('mousemove', function(event) {
      tooltip.style('top', (event.pageY - 10) + 'px')
        .style('left', (event.pageX + 10) + 'px');
    })
    .on('mouseout', function() {
      tooltip.style('visibility', 'hidden');
    });

  simulation.on('tick', () => {
    updateShadows()
    // 限制节点在视野内
    nodes.forEach(d => {
      d.x = Math.max(20, Math.min(width - 20, d.x))
      d.y = Math.max(20, Math.min(height - 20, d.y))
    })

    // 计算节点边界交点（为让箭头贴合线尾，需再往外延伸 marker 宽度/2）
    function getNodeBoundaryPoint(node, x0, y0, isTarget = false) {
      // x0, y0 是外部点（连线另一端）
      // isTarget: true 表示是终点（需要为 marker 预留空间）
      const markerLen = 5; // marker 宽度
      if (node.level === 1) {
        // 圆
        const r = Math.max(36, node._textW/2 + 14);
        const dx = node.x - x0, dy = node.y - y0;
        const len = Math.sqrt(dx*dx + dy*dy);
        if (len === 0) return { x: node.x, y: node.y };
        let rr = r;
        if (isTarget) rr += markerLen/2;
        return {
          x: node.x - dx/len * rr,
          y: node.y - dy/len * rr
        };
      } else {
        // 矩形
        const w = node._textW + 32;
        const h = 38;
        const cx = node.x, cy = node.y;
        const dx = x0 - cx, dy = y0 - cy;
        const absDx = Math.abs(dx), absDy = Math.abs(dy);
        let scale = 0.5 / Math.max(absDx / w, absDy / h);
        scale = Math.min(scale, 1);
        // 终点再往外延伸 marker
        if (isTarget && (absDx > 1e-6 || absDy > 1e-6)) {
          const len = Math.sqrt(dx*dx + dy*dy);
          const extend = markerLen/2 / len;
          scale += extend;
        }
        return {
          x: cx + dx * scale,
          y: cy + dy * scale
        };
      }
    }

    // 连线端点适配圆/矩形，保证有最小长度和箭头
    const minLen = 40;
    link
      .attr('x1', d => {
        const s = d.source, t = d.target;
        const dx = t.x - s.x, dy = t.y - s.y;
        const len = Math.sqrt(dx*dx + dy*dy);
        if (len < 1e-6) return s.x;
        // 起点
        let pt = getNodeBoundaryPoint(s, t.x, t.y, false);
        // 保证最小长度
        if (len < minLen) {
          pt.x = s.x + (dx/len) * (len/2 - 2);
          pt.y = s.y + (dy/len) * (len/2 - 2);
        }
        return pt.x;
      })
      .attr('y1', d => {
        const s = d.source, t = d.target;
        const dx = t.x - s.x, dy = t.y - s.y;
        const len = Math.sqrt(dx*dx + dy*dy);
        if (len < 1e-6) return s.y;
        let pt = getNodeBoundaryPoint(s, t.x, t.y, false);
        if (len < minLen) {
          pt.x = s.x + (dx/len) * (len/2 - 2);
          pt.y = s.y + (dy/len) * (len/2 - 2);
        }
        return pt.y;
      })
      .attr('x2', d => {
        const s = d.source, t = d.target;
        const dx = s.x - t.x, dy = s.y - t.y;
        const len = Math.sqrt(dx*dx + dy*dy);
        if (len < 1e-6) return t.x;
        let pt = getNodeBoundaryPoint(t, s.x, s.y, true);
        if (len < minLen) {
          pt.x = t.x + (dx/len) * (len/2 - 2);
          pt.y = t.y + (dy/len) * (len/2 - 2);
        }
        return pt.x;
      })
      .attr('y2', d => {
        const s = d.source, t = d.target;
        const dx = s.x - t.x, dy = s.y - t.y;
        const len = Math.sqrt(dx*dx + dy*dy);
        if (len < 1e-6) return t.y;
        let pt = getNodeBoundaryPoint(t, s.x, s.y, true);
        if (len < minLen) {
          pt.x = t.x + (dx/len) * (len/2 - 2);
          pt.y = t.y + (dy/len) * (len/2 - 2);
        }
        return pt.y;
      });

    // 一级节点圆
    nodeGroup.selectAll('circle')
      .filter(d => d)
      .attr('cx', d => d.x)
      .attr('cy', d => d.y)
    // 其他节点矩形
    nodeGroup.selectAll('rect')
      .filter(d => d)
      .attr('x', d => d.x - (d._textW + 32)/2)
      .attr('y', d => d.y - 19)
    label
      .attr('x', d => d.x)
      .attr('y', d => d.y)
  })
})
</script>