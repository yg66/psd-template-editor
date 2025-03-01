<template>
  <div>
    <input type="file" @change="handleFileUpload" accept=".psd"/>
    <!--    <div v-if="layers.length">-->
    <!--      <div v-for="(layer, index) in layers" :key="index" class="layer-item">-->
    <!--        <div>图层名: {{ layer.name }}</div>-->
    <!--        <div>位置: (水平位置【{{ layer.left }}】, 垂直位置【{{ layer.top }}】)</div>-->
    <!--        <div>尺寸: (图层宽度【{{ layer.right - layer.left }}】, 图层高度【{{ layer.bottom - layer.top }}】)</div>-->
    <!--        <div>可见性: {{ layer.visible ? '可见' : '隐藏' }}</div>-->
    <!--        <div v-if="layer.children.length" class="children">-->
    <!--          <div v-for="(child, i) in layer.children" :key="i" class="child">-->
    <!--            {{ child.name }}-->
    <!--          </div>-->
    <!--        </div>-->
    <!--      </div>-->
    <!--    </div>-->
    <div v-if="layers.length" class="layer-tree">
      <layer-item
          v-for="layer in layers"
          :key="layer.name"
          :layer="layer"
          :depth="0"
      />
    </div>
  </div>
</template>

<script setup>
import {ref} from 'vue';
import * as parsePsd from 'ag-psd';
import LayerItem from "@/components/LayerItem.vue";

const layers = ref([]);

// 递归提取图层信息（含子图层）
const extractLayers = (layers, depth = 0) => {
  // 添加层级缩进标识
  const indent = '│ '.repeat(depth);
  return layers.map(layer => {
    var layerType = '';
    if ('children' in layer) {
      // group
      layerType = 'children';
    } else if ('text' in layer) {
      // text layer
      layerType = 'text';
    } else if ('adjustment' in layer) {
      // adjustment layer
      layerType = 'adjustment';
    } else if ('placedLayer' in layer) {
      // smart object layer
      layerType = 'placedLayer';
    } else if ('vectorMask' in layer) {
      // vector layer
      layerType = 'vectorMask';
    } else {
      // bitmap layer
      layerType = 'bitmap';
    }

    // console.groupCollapsed(`${indent}图层 "${layer.name || '未命名'}"`);
    // 带缩进的日志输出
    console.log(`${indent}┌─ 图层: "${layer.name}" [L${depth}]`);
    console.log(`${indent}├─ 图层边界框: (${layer.left},${layer.top}) -> (${layer.right - layer.left},${layer.bottom - layer.top})`);
    console.log(`${indent}├─ 原始数据: `, layer);
    console.log(`${indent}├─ 是否隐藏: `, !layer.hidden ? '可见' : '隐藏');

    // 如果有蒙版等特殊属性
    if (layer.mask) {
      const markInfo = {
        maskTop: layer.mask.top,
        maskBottom: layer.mask.bottom,
        maskLeft: layer.mask.left,
        maskRight: layer.mask.right
      }
      console.log(`${indent}├─ 蒙版信息: `, markInfo);
    }
    console.log(`${indent}└─ 类型: `, layerType);

    // console.groupEnd();

    // 递归处理子图层时增加层级
    const children = layer.children ? extractLayers(layer.children, depth + 1) : [];

    return {
      name: layer.name || `未命名图层_${depth}_${Math.random().toString(36).substr(2, 5)}`,
      depth, // 添加深度标识
      top: layer.top,
      left: layer.left,
      right: layer.right,
      bottom: layer.bottom,
      visible: !layer.hidden,
      layerType: layerType,
      children
    };
  });
};

const handleFileUpload = (event) => {
  const file = event.target.files[0];
  if (!file) return;

  const reader = new FileReader();
  reader.onload = (e) => {
    try {
      const psd = parsePsd.readPsd(e.target.result);
      layers.value = extractLayers(psd.children || []);
    } catch (err) {
      console.error('解析失败:', err);
      alert('文件解析失败，请确认是有效的PSD文件');
    }
  };
  reader.readAsArrayBuffer(file);
};
</script>

<style>
.layer-item {
  margin: 10px;
  padding: 10px;
  border: 1px solid #ccc;
}

.children {
  margin-left: 20px;
  border-left: 2px solid #999;
  padding-left: 10px;
}

.child {
  color: #666;
  font-size: 0.9em;
}
</style>
