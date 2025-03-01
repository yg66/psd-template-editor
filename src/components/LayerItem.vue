<!-- LayerItem.vue -->
<template>
  <div class="layer-node" :style="{ marginLeft: `${depth * 20}px` }">
    <div class="layer-header">
      <span v-if="hasChildren" @click="toggle" class="toggle-icon">
        {{ isOpen ? '▼' : '▶' }}
      </span>
      {{ layer.name }}
      <div class="layer-meta">
        位置: (水平位置【{{ layer.left }}】, 垂直位置【{{ layer.top }}】)<br/>
        尺寸: (图层宽度【{{ layer.right - layer.left }}】, 图层高度【{{ layer.bottom - layer.top }}】)<br/>
        可见性: {{ layer.visible ? '可见' : '隐藏' }}<br/>
        类型: {{ layer.layerType }}
      </div>
    </div>

    <div v-show="isOpen" v-if="hasChildren">
      <layer-item
          v-for="child in layer.children"
          :key="child.name"
          :layer="child"
          :depth="depth + 1"
      />
    </div>
  </div>
</template>

<script setup>
import {ref, computed} from 'vue';

const props = defineProps({
  layer: Object,
  depth: Number
});

const isOpen = ref(true);
const hasChildren = computed(() => props.layer.children?.length > 0);

const toggle = () => {
  isOpen.value = !isOpen.value;
};
</script>

<style>
.layer-node {
  margin: 5px 0;
  padding: 8px;
  border-left: 2px solid #eee;
  transition: all 0.3s;
}

.toggle-icon {
  cursor: pointer;
  margin-right: 5px;
}

.layer-header:hover {
  background-color: #f5f5f5;
}

.layer-meta {
  color: #666;
  font-size: 0.8em;
}
</style>
