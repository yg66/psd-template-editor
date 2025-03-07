<template>
    <div class="psd-editor">
        <!-- 文件上传 -->
        <div class="file-upload">
            <input type="file" @change="handleFileChange" accept=".psd" />
            <button @click="exportPsd" :disabled="!psdData">导出PSD</button>
        </div>

        <div class="editor-container" v-if="psdData">
            <!-- 左侧图层面板 -->
            <div class="layers-panel">
                <h3>图层列表</h3>
                <div class="layer-tree">
                    <psd-layer-node :key="refreshKey" :layer="psdData" :depth="0" :selected-layer="selectedLayer"
                        @select-layer="selectLayer" />
                </div>
            </div>

            <!-- 中间预览区域 -->
            <div class="preview-area">
                <div class="psd-preview" ref="previewContainer" :style="previewStyle" @click="handleCanvasClick">
                    <psd-layer-preview :key="refreshKey" :layer="psdData" :parent-hidden="false"
                        :selected-layer="selectedLayer" @select-layer="selectLayer" />
                </div>
            </div>

            <!-- 右侧属性面板 -->
            <div class="properties-panel" v-if="selectedLayer">
                <h3>图层属性</h3>

                <!-- 裁剪状态提示 -->
                <div v-if="selectedLayer.needsClipping" class="clipping-warning">
                    <span class="warning-icon">⚠️</span>
                    <span>此图层超出边界，在预览中被裁剪</span>
                </div>

                <div class="property-group">
                    <div class="property-item">
                        <label>名称:</label>
                        <input v-model="selectedLayer.name" @change="updateLayer" />
                    </div>

                    <div class="property-item">
                        <label>可见性:</label>
                        <input type="checkbox" :checked="!selectedLayer.hidden" @change="toggleVisibility" />
                    </div>

                    <div class="property-item">
                        <label>不透明度:</label>
                        <input type="range" min="0" max="1" step="0.01" v-model.number="selectedLayer.opacity"
                            @change="updateLayer" />
                        <span>{{ Math.round(selectedLayer.opacity * 100) }}%</span>
                    </div>

                    <div class="property-item">
                        <label>混合模式:</label>
                        <select v-model="selectedLayer.blendMode" @change="updateLayer">
                            <option value="normal">正常</option>
                            <option value="multiply">正片叠底</option>
                            <option value="screen">滤色</option>
                            <option value="overlay">叠加</option>
                            <option value="darken">变暗</option>
                            <option value="lighten">变亮</option>
                            <option value="color-dodge">颜色减淡</option>
                            <option value="color-burn">颜色加深</option>
                            <option value="hard-light">强光</option>
                            <option value="soft-light">柔光</option>
                            <option value="difference">差值</option>
                            <option value="exclusion">排除</option>
                            <option value="hue">色相</option>
                            <option value="saturation">饱和度</option>
                            <option value="color">颜色</option>
                            <option value="luminosity">亮度</option>
                        </select>
                    </div>

                    <!-- 文本图层特有属性 -->
                    <div class="property-group" v-if="isTextLayer">
                        <h4>文本属性</h4>
                        <div class="property-item">
                            <label>文本内容:</label>
                            <textarea v-model="selectedLayer.text.text" @change="updateTextLayer"></textarea>
                        </div>
                        <div class="property-item">
                            <label>字体:</label>
                            <select v-model="selectedLayer.text.font.name" @change="updateTextLayer">
                                <option value="Arial">Arial</option>
                                <option value="Helvetica">Helvetica</option>
                                <option value="Times New Roman">Times New Roman</option>
                                <option value="SimSun">宋体</option>
                                <option value="Microsoft YaHei">微软雅黑</option>
                            </select>
                        </div>
                        <div class="property-item">
                            <label>字体大小:</label>
                            <input type="number" v-model.number="selectedLayer.text.font.sizes[0]"
                                @change="updateTextLayer" />
                        </div>
                        <div class="property-item">
                            <label>颜色:</label>
                            <input type="color" v-model="textColor" @change="updateTextColor" />
                        </div>
                    </div>

                    <!-- 位置和尺寸 -->
                    <div class="property-group">
                        <h4>位置和尺寸</h4>
                        <div class="property-item">
                            <label>X:</label>
                            <input type="number" v-model.number="selectedLayer.left" @change="updatePosition" />
                        </div>
                        <div class="property-item">
                            <label>Y:</label>
                            <input type="number" v-model.number="selectedLayer.top" @change="updatePosition" />
                        </div>
                        <div class="property-item">
                            <label>宽度:</label>
                            <input type="number" v-model.number="selectedLayer.width" @change="updateDimensions" />
                        </div>
                        <div class="property-item">
                            <label>高度:</label>
                            <input type="number" v-model.number="selectedLayer.height" @change="updateDimensions" />
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>

<script setup>
import { ref, computed, defineComponent, nextTick } from 'vue';
import { readPsd, writePsd } from 'ag-psd';

// 图层树形节点组件
const PsdLayerNode = defineComponent({
    name: 'PsdLayerNode',
    props: {
        layer: Object,
        depth: Number,
        selectedLayer: Object
    },
    emits: ['select-layer'],
    template: `
    <div 
      class="layer-node" 
      :style="{ marginLeft: depth * 15 + 'px' }"
      :class="{ 
        'selected': layer === selectedLayer,
        'clipped': layer.needsClipping === true && !layer.isRoot
      }"
      @click.stop="$emit('select-layer', layer)"
    >
      <div class="layer-header">
        <span class="visibility-toggle" @click.stop="toggleVisibility">
          {{ layer.hidden ? '👁️‍🗨️' : '👁️' }}
        </span>
        <span class="layer-name">{{ layer.name }}</span>
        <span class="layer-type">({{ getLayerType(layer) }})</span>
        <span v-if="layer.needsClipping" class="layer-clipped" title="此图层超出边界，已被裁剪">🔒</span>
      </div>
      <template v-if="layer.children">
        <psd-layer-node
          v-for="child in layer.children"
          :key="child.id"
          :layer="child"
          :depth="depth + 1"
          :selected-layer="selectedLayer"
          @select-layer="$emit('select-layer', $event)"
        />
      </template>
    </div>
  `,
    methods: {
        getLayerType(layer) {
            if ('children' in layer) return '组';
            if ('text' in layer) return '文本';
            if ('adjustment' in layer) return '调整';
            if ('placedLayer' in layer) return '智能对象';
            if ('vectorMask' in layer) return '矢量';
            return '位图';
        },
        toggleVisibility() {
            this.layer.hidden = !this.layer.hidden;
        }
    }
});

// 图层预览组件
const PsdLayerPreview = defineComponent({
    name: 'PsdLayerPreview',
    props: {
        layer: Object,
        parentHidden: Boolean,
        selectedLayer: Object
    },
    emits: ['select-layer'],
    template: `
    <div>
      <div
        v-if="!isHidden && hasContent(layer) && !shouldSkipRender(layer)"
        class="layer-element"
        :class="{ 'selected': layer === selectedLayer }"
        :style="layerStyle(layer)"
        @click.stop="$emit('select-layer', layer)"
      >
        <img v-if="layer.imageData" :src="layer.imageData" :style="imageStyle(layer)" @load="onImageLoad"/>
        <canvas v-else-if="layer.canvas" ref="canvas"/>
        <div v-else-if="layer.text" class="text-layer" :style="textStyle(layer)">{{ layer.text.text }}</div>
        <div v-else class="empty-layer">{{ layer.name }}</div>
      </div>
      <template v-if="layer.children">
        <psd-layer-preview
          v-for="child in layer.children"
          :key="child.id"
          :layer="child"
          :parent-hidden="isHidden"
          :selected-layer="selectedLayer"
          @select-layer="$emit('select-layer', $event)"
        />
      </template>
    </div>
  `,
    computed: {
        isHidden() {
            return this.parentHidden || this.layer.hidden === true;
        }
    },
    methods: {
        hasContent(layer) {
            return layer.canvas || layer.imageData || layer.fillColor || layer.text;
        },
        shouldSkipRender(layer) {
            // 如果图层被标记为需要裁剪且不是根图层，则跳过渲染
            // 这是关键修复：完全跳过超出边界的图层渲染
            return layer.needsClipping === true && !layer.isRoot;
        },
        layerStyle(layer) {
            const style = {
                position: 'absolute',
                left: Math.floor(layer.left || 0) + 'px',
                top: Math.floor(layer.top || 0) + 'px',
                width: Math.max(0, Math.floor(layer.width || 0)) + 'px',
                height: Math.max(0, Math.floor(layer.height || 0)) + 'px',
                opacity: (layer.opacity ?? 1),
                display: 'block',
                backgroundColor: layer.fillColor || 'transparent',
                zIndex: layer.zIndex || 0,
                mixBlendMode: this.getBlendMode(layer),
                border: layer === this.selectedLayer ? '2px dashed #00a8ff' : 'none',
                boxSizing: 'border-box',
                cursor: 'pointer',
                pointerEvents: 'auto',
                overflow: 'hidden'
            };

            // 特殊处理智能对象
            if (layer.isSmartObject) {
                // 确保智能对象不超出其边界
                style.overflow = 'hidden';
                style.clipPath = 'inset(0)'; // 裁剪到图层边界内
            }

            return style;
        },
        imageStyle(layer) {
            const style = {
                width: '100%',
                height: '100%',
                objectFit: 'contain',
                imageRendering: 'pixelated'
            };

            if (layer.placedLayer) {
                // 智能对象需要特殊处理
                style.objectFit = 'cover'; // 使用cover可能更适合智能对象

                // 关键修复：添加clip属性确保内容不超出边界
                style.clipPath = 'inset(0)'; // 裁剪到图层边界内
                style.maxWidth = '100%';
                style.maxHeight = '100%';
            }

            return style;
        },
        textStyle(layer) {
            if (!layer.text || !layer.text.font) return {};

            const fontFamily = layer.text.font.name || 'Arial';
            const fontSize = (layer.text.font.sizes && layer.text.font.sizes[0]) || 12;
            const color = layer.text.color ?
                `rgb(${layer.text.color[0]}, ${layer.text.color[1]}, ${layer.text.color[2]})` :
                '#000000';

            return {
                fontFamily,
                fontSize: `${fontSize}px`,
                color,
                lineHeight: '1.2',
                whiteSpace: 'pre-wrap',
                wordBreak: 'break-word'
            };
        },
        getBlendMode(layer) {
            const blendModeMap = {
                'normal': 'normal',
                'multiply': 'multiply',
                'screen': 'screen',
                'overlay': 'overlay',
                'darken': 'darken',
                'lighten': 'lighten',
                'colordodge': 'color-dodge',
                'colorburn': 'color-burn',
                'hard-light': 'hard-light',
                'soft-light': 'soft-light',
                'difference': 'difference',
                'exclusion': 'exclusion',
                'hue': 'hue',
                'saturation': 'saturation',
                'color': 'color',
                'luminosity': 'luminosity',

                'norm': 'normal',
                'mul': 'multiply',
                'scrn': 'screen',
                'over': 'overlay',
                'dark': 'darken',
                'lite': 'lighten',
                'cdod': 'color-dodge',
                'cbrn': 'color-burn',
                'hlit': 'hard-light',
                'slit': 'soft-light',
                'diff': 'difference',
                'smud': 'exclusion',
                'colr': 'color',
                'lum': 'luminosity'
            };

            const mode = (layer.blendMode || 'normal').toLowerCase();
            return blendModeMap[mode] || 'normal';
        },
        onImageLoad(event) {
            // 当图像加载完成时，发送到控制台以便调试
            if (this.layer && this.layer.imageData) {
                console.log(`图层 "${this.layer.name}" 的图像已加载`);
                sendImageToConsole(this.layer.imageData, `预览组件中的图层 "${this.layer.name}"`);
            }
        }
    },
    mounted() {
        if (this.layer.canvas && this.$refs.canvas) {
            const canvas = this.$refs.canvas;
            canvas.width = this.layer.width || 0;
            canvas.height = this.layer.height || 0;
            const ctx = canvas.getContext('2d');

            // 确保layer.canvas是有效对象且具有有效尺寸
            if (ctx && typeof this.layer.canvas === 'object' && this.layer.canvas !== null &&
                'width' in this.layer.canvas && 'height' in this.layer.canvas &&
                this.layer.canvas.width > 0 && this.layer.canvas.height > 0) {
                try {
                    // 特殊处理智能对象
                    if (this.layer.isSmartObject) {
                        // 智能对象可能需要特殊的绘制方式
                        // 确保内容不超出边界
                        ctx.save();
                        ctx.beginPath();
                        ctx.rect(0, 0, canvas.width, canvas.height);
                        ctx.clip();

                        // 绘制智能对象内容
                        ctx.drawImage(
                            this.layer.canvas,
                            0, 0, this.layer.canvas.width, this.layer.canvas.height,
                            0, 0, canvas.width, canvas.height
                        );

                        ctx.restore();
                    } else {
                        // 普通图层的绘制
                        ctx.drawImage(
                            this.layer.canvas,
                            0, 0, this.layer.canvas.width, this.layer.canvas.height,
                            0, 0, canvas.width, canvas.height
                        );
                    }

                    // 将canvas转换为图像数据并发送到控制台
                    const imageData = canvas.toDataURL('image/png');
                    sendImageToConsole(imageData, `预览组件中的Canvas "${this.layer.name}"`);
                } catch (e) {
                    console.error('Canvas渲染失败:', e, this.layer);
                }
            } else {
                console.warn(`图层 "${this.layer.name}" 的Canvas无效或尺寸无效，跳过渲染`);
                // 如果有图像数据，可以尝试使用它
                if (this.layer.imageData) {
                    console.log(`图层 "${this.layer.name}" 有图像数据，将使用图像数据显示`);
                    // 创建一个临时图像元素来替代canvas
                    const img = new Image();
                    img.src = this.layer.imageData;
                    img.style.width = '100%';
                    img.style.height = '100%';
                    img.style.objectFit = 'contain';

                    // 清空并替换canvas
                    canvas.parentNode.replaceChild(img, canvas);

                    // 发送图像数据到控制台
                    sendImageToConsole(this.layer.imageData, `预览组件中的替代图像 "${this.layer.name}"`);
                }
            }
        }
    }
});

// 主组件逻辑
const psdData = ref(null);
const selectedLayer = ref(null);
const previewContainer = ref(null);
const previewStyle = ref({});
const textColor = ref('#000000');
const refreshKey = ref(0); // 添加刷新键，用于强制组件重新渲染

// 计算属性
const isTextLayer = computed(() => {
    return selectedLayer.value && 'text' in selectedLayer.value;
});

// 使用更可靠的ID生成方式
let layerIdCounter = 0;
const generateLayerId = () => `layer_${layerIdCounter++}`;

// 添加一个辅助函数，用于发送图像数据到控制台
const sendImageToConsole = (imageData, layerName) => {
    try {
        // 创建一个临时的fetch请求，这样在网络面板中可以看到图像
        // 注意：这个请求会失败，但图像数据会显示在网络面板中
        fetch(imageData)
            .catch(err => {
                console.log(`图层 "${layerName}" 的图像数据已发送到网络面板`);
            });
    } catch (e) {
        console.error('发送图像到控制台失败:', e);
    }
};

// 处理文件上传
const handleFileChange = async (e) => {
    const file = e.target.files[0];
    if (!file) return;

    try {
        const arrayBuffer = await file.arrayBuffer();
        const psd = readPsd(arrayBuffer, {
            skipLayerImageData: false,
            skipCompositeImageData: false,
            logMissingFeatures: true
        });

        console.log("PSD基本信息:", {
            宽度: psd.width,
            高度: psd.height,
            图层数量: psd.children ? psd.children.length : 0,
            颜色模式: psd.colorMode
        });

        processPsdLayers(psd);
        console.log("PSD解析成功:", psd);
        psdData.value = psd;
        setupPreview(psd);
        selectedLayer.value = null; // 重置选中图层
    } catch (error) {
        console.error('PSD解析失败:', error);
        alert('PSD文件解析失败，请检查文件格式是否正确');
    }
};

// 处理图层数据
const processPsdLayers = (layer, zIndex = 1000, parentPath = '', parentBounds = null) => {
    if (!layer) return;

    // 生成图层路径，用于调试
    const layerPath = parentPath ? `${parentPath} > ${layer.name || '未命名'}` : (layer.name || '根图层');

    // 确保基本属性存在
    layer.opacity = typeof layer.opacity === 'number' ? layer.opacity : 1;

    // 处理混合模式 - 确保使用正确的混合模式名称
    if (layer.blendMode) {
        // 标准化混合模式名称
        const blendModeMap = {
            'norm': 'normal',
            'mul': 'multiply',
            'scrn': 'screen',
            'over': 'overlay',
            'dark': 'darken',
            'lite': 'lighten',
            'cdod': 'colorDodge',
            'cbrn': 'colorBurn',
            'hLit': 'hardLight',
            'sLit': 'softLight',
            'diff': 'difference',
            'smud': 'exclusion',
            'hue': 'hue',
            'sat': 'saturation',
            'colr': 'color',
            'lum': 'luminosity'
        };

        // 如果是简写形式，转换为完整形式
        layer.blendMode = blendModeMap[layer.blendMode] || layer.blendMode;
    } else {
        layer.blendMode = 'normal';
    }

    // 设置z-index用于图层叠加顺序
    layer.zIndex = zIndex;

    // 确保图层有唯一ID
    layer.id = layer.id || generateLayerId();

    // 处理图层尺寸
    if (layer.left !== undefined && layer.right !== undefined) {
        layer.width = layer.width || (layer.right - layer.left);
        layer.height = layer.height || (layer.bottom - layer.top);

        // 确保尺寸为正数
        layer.width = Math.max(0, layer.width);
        layer.height = Math.max(0, layer.height);

        // 记录当前图层的边界
        const currentBounds = {
            left: layer.left,
            top: layer.top,
            right: layer.left + layer.width,
            bottom: layer.top + layer.height
        };

        // 如果有父边界，检查当前图层是否超出
        if (parentBounds && !layer.isRoot) {
            // 检查是否超出父边界
            const isOutOfBounds =
                currentBounds.right > parentBounds.right ||
                currentBounds.bottom > parentBounds.bottom ||
                currentBounds.left < parentBounds.left ||
                currentBounds.top < parentBounds.top;

            if (isOutOfBounds) {
                console.warn(`图层 "${layerPath}" 超出父边界，将被裁剪`);

                // 标记为需要裁剪
                layer.needsClipping = true;
            } else {
                // 确保不再被标记为需要裁剪
                layer.needsClipping = false;
            }
        }
    }

    // 特殊处理智能对象图层
    if (layer.placedLayer) {
        console.log(`处理智能对象图层 "${layerPath}":`, layer.placedLayer);

        // 标记为智能对象，以便在渲染时特殊处理
        layer.isSmartObject = true;

        // 如果智能对象超出主体边界，标记为需要裁剪
        if (parentBounds && !layer.isRoot) {
            const isOutOfBounds =
                (layer.left + layer.width) > parentBounds.right ||
                (layer.top + layer.height) > parentBounds.bottom ||
                layer.left < parentBounds.left ||
                layer.top < parentBounds.top;

            if (isOutOfBounds) {
                console.warn(`智能对象图层 "${layerPath}" 超出父边界，将被裁剪`);
                layer.needsClipping = true;
            } else {
                // 确保不再被标记为需要裁剪
                layer.needsClipping = false;
            }
        }

        // 如果智能对象有自己的尺寸，确保它不超出图层边界
        if (layer.canvas) {
            try {
                // 确保canvas有效
                if (layer.canvas.width > 0 && layer.canvas.height > 0) {
                    layer.imageData = layer.canvas.toDataURL('image/png');
                    console.log(`智能对象图层 "${layerPath}" 的Canvas已转换为图像数据`);

                    // 发送图像到控制台
                    sendImageToConsole(layer.imageData, layerPath);

                    // 记录原始尺寸
                    layer.originalWidth = layer.canvas.width;
                    layer.originalHeight = layer.canvas.height;
                }
            } catch (e) {
                console.error(`智能对象图层 "${layerPath}" 的Canvas转换失败:`, e);
            }
        }
    }

    // 处理图层内容
    if (layer.canvas) {
        try {
            // 确保canvas是有效对象且具有有效尺寸
            if (typeof layer.canvas === 'object' && layer.canvas !== null &&
                'width' in layer.canvas && 'height' in layer.canvas &&
                layer.canvas.width > 0 && layer.canvas.height > 0) {
                layer.imageData = layer.canvas.toDataURL('image/png');
                console.log(`图层 "${layerPath}" 的Canvas已转换为图像数据`);

                // 发送图像到控制台
                sendImageToConsole(layer.imageData, layerPath);
            } else {
                console.warn(`图层 "${layerPath}" 的Canvas无效或尺寸无效: ${layer.canvas?.width}x${layer.canvas?.height}`);
                // 如果canvas无效，确保不会在后续代码中使用它
                if (layer.imageData) {
                    console.log(`图层 "${layerPath}" 已有图像数据，将继续使用`);

                    // 发送现有图像到控制台
                    sendImageToConsole(layer.imageData, layerPath + " (现有)");
                } else {
                    console.log(`图层 "${layerPath}" 没有有效的图像数据`);
                }
            }
        } catch (e) {
            console.error(`图层 "${layerPath}" 的Canvas转换失败:`, e);
        }
    }

    // 处理文本图层
    if (layer.text) {
        console.log(`处理文本图层 "${layerPath}":`, layer.text);

        // 确保文本图层有必要的属性
        layer.text.font = layer.text.font || {};
        layer.text.font.sizes = layer.text.font.sizes || [12];
        layer.text.font.name = layer.text.font.name || 'Arial';

        // 提取文本颜色
        if (layer.text.color) {
            const [r, g, b] = layer.text.color;
            textColor.value = rgbToHex(r, g, b);
        }

        // 如果文本图层被标记为已更新，需要重新生成图像
        if (layer.textUpdated) {
            // 这里可以添加重新生成文本图像的代码
            // 例如，使用canvas绘制文本
            console.log(`文本图层 "${layerPath}" 已更新，需要重新生成图像`);

            // 创建一个新的canvas来绘制文本
            if (!layer.canvas || layer.canvas.width <= 0 || layer.canvas.height <= 0) {
                // 如果没有有效的canvas，创建一个新的
                const canvas = document.createElement('canvas');
                // 设置canvas尺寸为图层尺寸
                canvas.width = layer.width || 200;  // 默认宽度
                canvas.height = layer.height || 100; // 默认高度
                layer.canvas = canvas;
            }

            // 获取canvas上下文
            const ctx = layer.canvas.getContext('2d');
            if (ctx) {
                // 清除画布
                ctx.clearRect(0, 0, layer.canvas.width, layer.canvas.height);

                // 设置文本样式
                const fontFamily = layer.text.font.name || 'Arial';
                const fontSize = (layer.text.font.sizes && layer.text.font.sizes[0]) || 12;
                const color = layer.text.color ?
                    `rgb(${layer.text.color[0]}, ${layer.text.color[1]}, ${layer.text.color[2]})` :
                    '#000000';

                ctx.font = `${fontSize}px ${fontFamily}`;
                ctx.fillStyle = color;
                ctx.textBaseline = 'top';

                // 绘制文本
                const text = layer.text.text || '';
                const lines = text.split('\n');
                const lineHeight = fontSize * 1.2;

                lines.forEach((line, index) => {
                    ctx.fillText(line, 0, index * lineHeight);
                });

                // 更新图像数据
                try {
                    layer.imageData = layer.canvas.toDataURL('image/png');
                    console.log(`文本图层 "${layerPath}" 的图像已重新生成`);

                    // 发送文本图层图像到控制台
                    sendImageToConsole(layer.imageData, layerPath + " (文本)");
                } catch (e) {
                    console.error(`文本图层 "${layerPath}" 的图像生成失败:`, e);
                }
            }

            // 清除更新标记
            delete layer.textUpdated;
        }
    }

    // 设置当前图层的边界，用于子图层的边界检查
    const currentLayerBounds = {
        left: layer.left || 0,
        top: layer.top || 0,
        right: (layer.left || 0) + (layer.width || 0),
        bottom: (layer.top || 0) + (layer.height || 0)
    };

    // 如果是根图层，标记一下
    if (!parentPath) {
        layer.isRoot = true;
    }

    // 递归处理子图层
    if (layer.children && layer.children.length > 0) {
        console.log(`处理 "${layerPath}" 的子图层，数量: ${layer.children.length}`);

        // 反转子图层顺序（PSD中图层顺序与显示顺序相反）
        // 注意：PSD中图层从上到下的顺序在数组中是从前到后
        const orderedChildren = [...layer.children].reverse();

        layer.children = orderedChildren.map((child, index) => {
            // 传递当前图层的边界给子图层，用于边界检查
            return processPsdLayers(child, zIndex - index - 1, layerPath, currentLayerBounds);
        });
    }

    return layer;
};

// 设置预览区域
const setupPreview = (psd) => {
    if (!psd || !psd.width || !psd.height) {
        console.error('PSD数据不完整，无法设置预览区域');
        return;
    }

    // 设置预览容器样式
    previewStyle.value = {
        width: psd.width + 'px',
        height: psd.height + 'px',
        position: 'relative',
        background: generateCheckerboard(),
        backgroundSize: '8px 8px',
        boxShadow: '0 0 10px rgba(0, 0, 0, 0.2)',
        margin: '0 auto'
    };

    // 如果有合成图像，可以作为背景显示
    if (psd.canvas) {
        try {
            const compositeImageUrl = psd.canvas.toDataURL('image/png');
            console.log('合成图像已生成');

            // 可以选择是否将合成图像作为背景
            // previewStyle.value.backgroundImage = `url(${compositeImageUrl})`;
            // previewStyle.value.backgroundSize = 'contain';
            // previewStyle.value.backgroundRepeat = 'no-repeat';

            // 更新PSD的合成图像
            psd.compositeImageData = compositeImageUrl;

            // 发送合成图像到控制台
            sendImageToConsole(psd.compositeImageData, "合成图像");
        } catch (e) {
            console.error('合成图像生成失败:', e);
        }
    }
};

// 生成棋盘背景
const generateCheckerboard = () => {
    const size = 8;
    const canvas = document.createElement('canvas');
    canvas.width = size * 2;
    canvas.height = size * 2;
    const ctx = canvas.getContext('2d');

    ctx.fillStyle = '#ddd';
    ctx.fillRect(0, 0, size * 2, size * 2);
    ctx.fillStyle = '#fff';
    ctx.fillRect(0, 0, size, size);
    ctx.fillRect(size, size, size, size);

    return `url(${canvas.toDataURL()})`;
};

// 处理画布点击事件
const handleCanvasClick = (event) => {
    // 如果没有PSD数据，直接返回
    if (!psdData.value) return;

    // 获取点击位置相对于预览容器的坐标
    const rect = previewContainer.value.getBoundingClientRect();
    const x = event.clientX - rect.left;
    const y = event.clientY - rect.top;

    // 查找点击位置的最上层图层
    const clickedLayer = findLayerAtPosition(psdData.value, x, y);
    if (clickedLayer) {
        selectLayer(clickedLayer);
    }
};

// 在指定位置查找图层
const findLayerAtPosition = (layer, x, y, visibleOnly = true) => {
    if (!layer) return null;

    // 如果图层被隐藏且只查找可见图层，则跳过
    if (visibleOnly && layer.hidden) return null;

    // 先检查子图层（从上到下）
    if (layer.children) {
        for (const child of layer.children) {
            const foundLayer = findLayerAtPosition(child, x, y, visibleOnly);
            if (foundLayer) return foundLayer;
        }
    }

    // 检查当前图层是否包含点击位置
    if (layer.left !== undefined && layer.top !== undefined &&
        layer.width !== undefined && layer.height !== undefined) {
        if (x >= layer.left && x <= layer.left + layer.width &&
            y >= layer.top && y <= layer.top + layer.height) {
            // 确保图层有内容
            if (layer.canvas || layer.imageData || layer.fillColor || layer.text) {
                return layer;
            }
        }
    }

    return null;
};

// 选择图层
const selectLayer = (layer) => {
    selectedLayer.value = layer;
    console.log('选中图层:', layer);

    // 如果是文本图层，更新文本颜色
    if (layer && layer.text && layer.text.color) {
        const [r, g, b] = layer.text.color;
        textColor.value = rgbToHex(r, g, b);
    }

    // 如果图层被标记为需要裁剪，显示提示信息
    if (layer && layer.needsClipping) {
        console.warn(`选中的图层 "${layer.name}" 超出边界，在预览中被裁剪`);
    }
};

// 更新图层属性
const updateLayer = () => {
    if (!selectedLayer.value) return;

    try {
        // 如果图层被标记为需要裁剪，可能需要重新评估
        if (selectedLayer.value.needsClipping) {
            // 查找父图层 - 直接使用递归查找，不依赖ID格式
            const findParentLayer = (root, targetLayer) => {
                if (!root || !root.children) return null;

                for (const child of root.children) {
                    if (child === targetLayer) return root;
                    const found = findParentLayer(child, targetLayer);
                    if (found) return found;
                }

                return null;
            };

            const parentLayer = findParentLayer(psdData.value, selectedLayer.value);

            if (parentLayer) {
                // 获取父图层边界
                const parentBounds = {
                    left: parentLayer.left || 0,
                    top: parentLayer.top || 0,
                    right: (parentLayer.left || 0) + (parentLayer.width || 0),
                    bottom: (parentLayer.top || 0) + (parentLayer.height || 0)
                };

                // 检查当前图层是否仍然超出父边界
                const currentBounds = {
                    left: selectedLayer.value.left || 0,
                    top: selectedLayer.value.top || 0,
                    right: (selectedLayer.value.left || 0) + (selectedLayer.value.width || 0),
                    bottom: (selectedLayer.value.top || 0) + (selectedLayer.value.height || 0)
                };

                const isOutOfBounds =
                    currentBounds.right > parentBounds.right ||
                    currentBounds.bottom > parentBounds.bottom ||
                    currentBounds.left < parentBounds.left ||
                    currentBounds.top < parentBounds.top;

                // 更新裁剪状态
                selectedLayer.value.needsClipping = isOutOfBounds;
            }
        }

        // 保存当前选中图层的引用
        const currentSelectedLayer = selectedLayer.value;

        // 在原始数据上重新处理图层数据，确保所有计算属性都是最新的
        processPsdLayers(psdData.value);

        // 重要：合成所有图层，生成新的预览图像
        const compositeImageData = composeLayers(psdData.value);

        // 更新预览区域
        if (compositeImageData) {
            // 更新预览容器的背景
            const currentStyle = { ...previewStyle.value };
            currentStyle.backgroundImage = `url(${compositeImageData})`;
            currentStyle.backgroundSize = 'contain';
            currentStyle.backgroundPosition = 'center';
            currentStyle.backgroundRepeat = 'no-repeat';

            previewStyle.value = currentStyle;
            console.log('预览区域已更新');

            // 发送合成图像到控制台
            sendImageToConsole(compositeImageData, "更新后的合成图像");
        }

        // 使用Vue的响应式系统来触发更新
        nextTick(() => {
            // 确保选中的图层仍然是当前图层
            selectedLayer.value = currentSelectedLayer;

            // 增加刷新键，强制组件重新渲染
            refreshKey.value++;
            console.log('图层属性已更新，预览已刷新，刷新键:', refreshKey.value);
        });
    } catch (error) {
        console.error('更新图层属性时发生错误:', error);
    }
};

// 切换图层可见性
const toggleVisibility = (event) => {
    if (!selectedLayer.value) return;

    try {
        // 更新可见性状态
        selectedLayer.value.hidden = !event.target.checked;

        // 直接调用更新图层函数
        updateLayer();
    } catch (error) {
        console.error('切换图层可见性时发生错误:', error);
    }
};

// 更新文本图层
const updateTextLayer = () => {
    if (!selectedLayer.value || !isTextLayer.value) return;

    try {
        // 标记文本图层已更新
        selectedLayer.value.textUpdated = true;

        // 对于文本图层，我们需要重新生成图像
        // 1. 清除现有的图像数据，这样processPsdLayers会重新生成
        delete selectedLayer.value.imageData;

        // 2. 如果有canvas，也需要清除或重新生成
        if (selectedLayer.value.canvas) {
            // 方法1：删除canvas，让processPsdLayers重新生成
            // delete selectedLayer.value.canvas;

            // 方法2：重新绘制canvas（更好的方法）
            const canvas = selectedLayer.value.canvas;
            const ctx = canvas.getContext('2d');
            if (ctx) {
                // 清除画布
                ctx.clearRect(0, 0, canvas.width, canvas.height);

                // 设置文本样式
                const fontFamily = selectedLayer.value.text.font.name || 'Arial';
                const fontSize = (selectedLayer.value.text.font.sizes && selectedLayer.value.text.font.sizes[0]) || 12;
                const color = selectedLayer.value.text.color ?
                    `rgb(${selectedLayer.value.text.color[0]}, ${selectedLayer.value.text.color[1]}, ${selectedLayer.value.text.color[2]})` :
                    '#000000';

                ctx.font = `${fontSize}px ${fontFamily}`;
                ctx.fillStyle = color;
                ctx.textBaseline = 'top';

                // 绘制文本（简单实现，可以根据需要扩展）
                const text = selectedLayer.value.text.text || '';
                const lines = text.split('\n');
                const lineHeight = fontSize * 1.2;

                lines.forEach((line, index) => {
                    ctx.fillText(line, 0, index * lineHeight);
                });

                // 更新图像数据
                selectedLayer.value.imageData = canvas.toDataURL('image/png');
                console.log('文本图层图像已重新生成');

                // 发送文本图层图像到控制台
                sendImageToConsole(selectedLayer.value.imageData, selectedLayer.value.name + " (文本更新)");
            }
        }

        // 直接调用更新图层函数
        updateLayer();
    } catch (error) {
        console.error('更新文本图层时发生错误:', error);
    }
};

// 更新文本颜色
const updateTextColor = () => {
    if (!selectedLayer.value || !isTextLayer.value) return;

    try {
        // 将十六进制颜色转换为RGB数组
        const rgb = hexToRgb(textColor.value);
        selectedLayer.value.text.color = [rgb.r, rgb.g, rgb.b];

        // 直接调用更新文本图层函数
        updateTextLayer();
    } catch (error) {
        console.error('更新文本颜色时发生错误:', error);
    }
};

// 更新图层位置
const updatePosition = () => {
    if (!selectedLayer.value) return;

    try {
        // 更新right和bottom值以保持一致性
        selectedLayer.value.right = selectedLayer.value.left + selectedLayer.value.width;
        selectedLayer.value.bottom = selectedLayer.value.top + selectedLayer.value.height;

        // 检查图层是否超出父边界
        checkLayerBounds();

        // 直接调用更新图层函数
        updateLayer();
    } catch (error) {
        console.error('更新图层位置时发生错误:', error);
    }
};

// 更新图层尺寸
const updateDimensions = () => {
    if (!selectedLayer.value) return;

    try {
        // 更新right和bottom值以保持一致性
        selectedLayer.value.right = selectedLayer.value.left + selectedLayer.value.width;
        selectedLayer.value.bottom = selectedLayer.value.top + selectedLayer.value.height;

        // 检查图层是否超出父边界
        checkLayerBounds();

        // 直接调用更新图层函数
        updateLayer();
    } catch (error) {
        console.error('更新图层尺寸时发生错误:', error);
    }
};

// 检查图层是否超出父边界
const checkLayerBounds = () => {
    if (!selectedLayer.value || selectedLayer.value.isRoot) return;

    try {
        // 查找父图层 - 直接使用递归查找，不依赖ID格式
        const findParentLayer = (root, targetLayer) => {
            if (!root || !root.children) return null;

            for (const child of root.children) {
                if (child === targetLayer) return root;
                const found = findParentLayer(child, targetLayer);
                if (found) return found;
            }

            return null;
        };

        const parentLayer = findParentLayer(psdData.value, selectedLayer.value);

        if (parentLayer) {
            // 获取父图层边界
            const parentBounds = {
                left: parentLayer.left || 0,
                top: parentLayer.top || 0,
                right: (parentLayer.left || 0) + (parentLayer.width || 0),
                bottom: (parentLayer.top || 0) + (parentLayer.height || 0)
            };

            // 检查当前图层是否超出父边界
            const currentBounds = {
                left: selectedLayer.value.left || 0,
                top: selectedLayer.value.top || 0,
                right: (selectedLayer.value.left || 0) + (selectedLayer.value.width || 0),
                bottom: (selectedLayer.value.top || 0) + (selectedLayer.value.height || 0)
            };

            const isOutOfBounds =
                currentBounds.right > parentBounds.right ||
                currentBounds.bottom > parentBounds.bottom ||
                currentBounds.left < parentBounds.left ||
                currentBounds.top < parentBounds.top;

            // 更新裁剪状态
            selectedLayer.value.needsClipping = isOutOfBounds;

            console.log(`图层 "${selectedLayer.value.name}" 边界检查:`,
                isOutOfBounds ? '超出边界，将被裁剪' : '在边界内，正常显示');
        }
    } catch (error) {
        console.error('检查图层边界时发生错误:', error);
    }
};

// 导出PSD文件
const exportPsd = async () => {
    if (!psdData.value) return;

    try {
        // 准备导出前的处理
        const psdForExport = JSON.parse(JSON.stringify(psdData.value));

        // 将图层顺序恢复为PSD格式所需的顺序
        if (psdForExport.children) {
            psdForExport.children = [...psdForExport.children].reverse();
        }

        // 生成PSD文件
        const buffer = writePsd(psdForExport, { generateThumbnail: true });

        // 创建Blob并下载
        const blob = new Blob([buffer], { type: 'application/octet-stream' });
        const url = URL.createObjectURL(blob);

        const a = document.createElement('a');
        a.href = url;
        a.download = 'edited.psd';
        document.body.appendChild(a);
        a.click();
        document.body.removeChild(a);
        URL.revokeObjectURL(url);

        console.log('PSD导出成功');
    } catch (error) {
        console.error('PSD导出失败:', error);
        alert('PSD导出失败，请查看控制台获取详细信息');
    }
};

// 辅助函数：RGB转十六进制
const rgbToHex = (r, g, b) => {
    return '#' + [r, g, b].map(x => {
        const hex = Math.round(x).toString(16);
        return hex.length === 1 ? '0' + hex : hex;
    }).join('');
};

// 辅助函数：十六进制转RGB
const hexToRgb = (hex) => {
    const result = /^#?([a-f\d]{2})([a-f\d]{2})([a-f\d]{2})$/i.exec(hex);
    return result ? {
        r: parseInt(result[1], 16),
        g: parseInt(result[2], 16),
        b: parseInt(result[3], 16)
    } : { r: 0, g: 0, b: 0 };
};

// 合成所有图层，生成预览图像
const composeLayers = (psd) => {
    if (!psd || !psd.width || !psd.height) {
        console.error('PSD数据不完整，无法合成图层');
        return;
    }

    console.log('开始合成所有图层...');

    // 创建一个新的canvas用于合成
    const compositeCanvas = document.createElement('canvas');
    compositeCanvas.width = psd.width;
    compositeCanvas.height = psd.height;
    const ctx = compositeCanvas.getContext('2d');

    // 清空画布，设置透明背景
    ctx.clearRect(0, 0, compositeCanvas.width, compositeCanvas.height);

    // 递归渲染图层
    const renderLayer = (layer) => {
        if (!layer || layer.hidden) return;

        // 如果是组，先渲染子图层
        if (layer.children && layer.children.length > 0) {
            // 从下到上渲染子图层（数组中是从后到前）
            for (let i = layer.children.length - 1; i >= 0; i--) {
                renderLayer(layer.children[i]);
            }
        }

        // 渲染当前图层
        if (layer.imageData && !layer.needsClipping) {
            try {
                // 创建图像对象
                const img = new Image();
                img.src = layer.imageData;

                // 同步绘制图像（这里使用了一个技巧，因为图像已经加载到浏览器缓存中）
                ctx.save();

                // 设置混合模式
                ctx.globalCompositeOperation = layer.blendMode || 'normal';

                // 设置不透明度
                ctx.globalAlpha = layer.opacity !== undefined ? layer.opacity : 1;

                // 绘制图像
                ctx.drawImage(
                    img,
                    layer.left || 0,
                    layer.top || 0,
                    layer.width || img.width,
                    layer.height || img.height
                );

                ctx.restore();
            } catch (e) {
                console.error(`图层 "${layer.name}" 渲染失败:`, e);
            }
        }
    };

    // 从根图层开始渲染
    renderLayer(psd);

    // 更新PSD的合成图像
    psd.compositeImageData = compositeCanvas.toDataURL('image/png');

    // 发送合成图像到控制台
    sendImageToConsole(psd.compositeImageData, "合成图像");

    // 如果需要，也可以保存canvas引用
    psd.compositeCanvas = compositeCanvas;

    console.log('图层合成完成');
    return psd.compositeImageData;
};
</script>

<style scoped>
.psd-editor {
    display: flex;
    flex-direction: column;
    height: 100%;
    width: 100%;
}

.file-upload {
    padding: 10px;
    background-color: #f5f5f5;
    border-bottom: 1px solid #ddd;
    display: flex;
    gap: 10px;
}

.editor-container {
    display: flex;
    flex: 1;
    overflow: hidden;
}

.layers-panel {
    width: 250px;
    border-right: 1px solid #ddd;
    overflow-y: auto;
    padding: 10px;
    background-color: white;
}

.preview-area {
    flex: 1;
    overflow: auto;
    padding: 20px;
    background-color: #f0f0f0;
    display: flex;
    justify-content: center;
    align-items: flex-start;
}

.psd-preview {
    box-shadow: 0 0 10px rgba(0, 0, 0, 0.2);
    image-rendering: pixelated;
    transform-origin: top left;
}

.properties-panel {
    width: 300px;
    border-left: 1px solid #ddd;
    overflow-y: auto;
    padding: 10px;
    background-color: white;
}

.layer-node {
    padding: 5px;
    border-bottom: 1px solid #eee;
    cursor: pointer;
}

.layer-node:hover {
    background-color: #f5f5f5;
}

.layer-node.selected {
    background-color: #e3f2fd;
}

.layer-header {
    display: flex;
    align-items: center;
    gap: 5px;
}

.visibility-toggle {
    cursor: pointer;
}

.layer-name {
    flex: 1;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
}

.layer-type {
    color: #888;
    font-size: 0.8em;
}

.layer-element {
    position: absolute;
    overflow: hidden;
    transform-origin: top left;
    box-sizing: border-box;
    clip-path: inset(0);
    /* 确保所有内容都被裁剪到图层边界内 */
}

.layer-element img {
    max-width: 100%;
    max-height: 100%;
    object-position: center;
}

.layer-element.selected {
    outline: 2px dashed #00a8ff;
    outline-offset: -2px;
}

.text-layer {
    white-space: pre-wrap;
    word-break: break-word;
    width: 100%;
    height: 100%;
    display: flex;
    align-items: center;
    overflow: hidden;
}

.empty-layer {
    width: 100%;
    height: 100%;
    display: flex;
    align-items: center;
    justify-content: center;
    background-color: rgba(200, 200, 200, 0.3);
    color: #666;
    font-style: italic;
    overflow: hidden;
}

.property-group {
    margin-bottom: 15px;
    padding: 10px;
    background-color: #f9f9f9;
    border-radius: 4px;
}

.property-group h4 {
    margin-top: 0;
    margin-bottom: 10px;
    font-size: 14px;
    color: #333;
}

.property-item {
    margin-bottom: 8px;
    display: flex;
    align-items: center;
}

.property-item label {
    width: 80px;
    font-size: 12px;
    color: #666;
}

.property-item input,
.property-item select,
.property-item textarea {
    flex: 1;
    padding: 4px;
    border: 1px solid #ddd;
    border-radius: 3px;
}

.property-item input[type="range"] {
    margin-right: 5px;
}

.property-item span {
    font-size: 12px;
    color: #666;
    width: 40px;
    text-align: right;
}

.layer-node.clipped {
    opacity: 0.7;
    background-color: #fff0f0;
}

.layer-clipped {
    color: #ff5252;
    margin-left: 5px;
    font-size: 12px;
}

.clipping-warning {
    margin-bottom: 15px;
    padding: 10px;
    background-color: #fff0f0;
    border-radius: 4px;
    display: flex;
    align-items: center;
    gap: 10px;
}

.warning-icon {
    color: #ff5252;
    font-size: 1.2em;
}
</style>