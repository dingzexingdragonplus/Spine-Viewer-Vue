<template>
    <div id="display">
        <div id="resolution" v-show="data.resolution.show">
            <div>Width: {{ data.resolution.width }}px</div>
            <div>Height:{{ data.resolution.height }}px</div>
        </div>
        <div id="coordinate" v-show="data.coordinate.show">
            <div>X: {{ data.coordinate.x.toFixed() }}px</div>
            <div>Y: {{ data.coordinate.y.toFixed() }}px</div>
        </div>
        <div id="spine-version" v-show="data.spineVersion">
            <span>Spine {{ data.spineVersion }}</span>
        </div>
    </div>
</template>

<script setup>
import {inject, onMounted, reactive, watch, onUnmounted} from "vue";
import {getById} from "@/utils/util";
import {useAppStore} from "@/stores/app";
import * as PIXI from "pixi.js";
import {Spine} from "pixi-spine";

const appStore = useAppStore()

const pixiApp = inject('pixiApp')

const data = reactive({
    resolution: {
        show: false,
        width: 0,
        height: 0
    },
    coordinate: {
        show: true,
        x: appStore.active.container.data.position.x,
        y: appStore.active.container.data.position.y
    },
    spineVersion: null
})

// 存储骨骼名称标签的容器
const boneLabelsContainers = new Map();

// 创建骨骼名称标签
function createBoneLabels(spine) {
    if (!spine || !spine.skeleton || !spine.skeleton.slots) return;
    
    // 如果已经为这个spine创建了标签，先清理
    if (boneLabelsContainers.has(spine)) {
        const oldData = boneLabelsContainers.get(spine);
        if (oldData.container.parent) {
            oldData.container.parent.removeChild(oldData.container);
        }
        oldData.container.destroy({ children: true });
        boneLabelsContainers.delete(spine);
    }
    
    // 创建一个新的容器来存放所有插槽名称标签
    const labelsContainer = new PIXI.Container();
    labelsContainer.visible = appStore.showBoneNames;
    labelsContainer.sortableChildren = true; // 启用排序
    labelsContainer.zIndex = 1000; // 确保标签显示在最上层
    
    // 为每个插槽创建文本标签
    spine.skeleton.slots.forEach(slot => {
        // 跳过没有名称的插槽
        if (!slot.data || !slot.data.name) return;
        
        const text = new PIXI.Text(slot.data.name, {
            fontFamily: 'Arial',
            fontSize: 12,
            fill: 0xffffff,
            align: 'center',
            stroke: 0x000000,
            strokeThickness: 3
        });
        
        text.anchor.set(0.5);
        // 获取插槽所属的骨骼
        const bone = slot.bone;
        if (!bone) return;
        
        // 计算插槽在世界坐标系中的位置
        const worldX = spine.x + bone.worldX * spine.scale.x;
        const worldY = spine.y + bone.worldY * spine.scale.y;
        text.position.set(worldX, worldY);
        labelsContainer.addChild(text);
    });
    
    // 将标签容器添加到spine的父容器中
    if (spine.parent) {
        spine.parent.addChild(labelsContainer);
        // 确保容器的排序
        if (typeof spine.parent.sortChildren === 'function') {
            spine.parent.sortChildren();
        }
    }
    
    // 存储标签容器的引用，以便后续更新
    boneLabelsContainers.set(spine, {
        container: labelsContainer,
        slots: spine.skeleton.slots
    });
    
    // 立即更新一次位置
    updateBoneLabelPosition(spine);
}

// 更新特定spine的插槽标签位置
function updateBoneLabelPosition(spine) {
    if (!boneLabelsContainers.has(spine)) return;

    const { container, slots } = boneLabelsContainers.get(spine);

    // 确保容器是可见的才进行位置更新和重排
    if (!container.visible) return;

    // 更新每个标签的位置
    const labelPositions = []; // 用于存储已放置标签的边界框
    let textIndex = 0;
    const padding = 5; // 标签之间的垂直间距

    slots.forEach(slot => {
        // 跳过没有名称的插槽或没有骨骼的插槽
        if (!slot.data || !slot.data.name || !slot.bone) return;

        if (textIndex < container.children.length) {
            const text = container.children[textIndex++];
            // 获取插槽所属的骨骼
            const bone = slot.bone;
            // 计算插槽在世界坐标系中的初始位置
            let worldX = spine.x + bone.worldX * spine.scale.x;
            let worldY = spine.y + bone.worldY * spine.scale.y;
            text.position.set(worldX, worldY);

            // 获取当前标签的边界框
            let currentBounds = text.getBounds();
            let overlaps = true;
            let attempts = 0; // 防止无限循环
            const maxAttempts = 20; // 最多尝试20次向下移动

            while (overlaps && attempts < maxAttempts) {
                overlaps = false;
                for (const placedBounds of labelPositions) {
                    // 简单的AABB碰撞检测
                    if (
                        currentBounds.x < placedBounds.x + placedBounds.width &&
                        currentBounds.x + currentBounds.width > placedBounds.x &&
                        currentBounds.y < placedBounds.y + placedBounds.height &&
                        currentBounds.y + currentBounds.height > placedBounds.y
                    ) {
                        overlaps = true;
                        // 如果重叠，将当前标签向下移动
                        worldY += currentBounds.height + padding;
                        text.position.set(worldX, worldY);
                        currentBounds = text.getBounds(); // 更新边界框
                        break; // 重新检查所有已放置的标签
                    }
                }
                attempts++;
            }
             // 如果尝试次数过多仍重叠，则将其移回原始位置（或提供其他回退逻辑）
             if (attempts >= maxAttempts && overlaps) {
                 console.warn(`Could not resolve overlap for label "${slot.data.name}" after ${maxAttempts} attempts.`);
                 worldY = spine.y + bone.worldY * spine.scale.y; // Reset to original Y
                 text.position.set(worldX, worldY);
                 currentBounds = text.getBounds(); // Reset bounds
            }


            // 添加当前标签的最终位置到已放置列表
            labelPositions.push(currentBounds);
        }
    });
}

// 更新骨骼名称标签的位置和可见性
function updateBoneLabels() {
    boneLabelsContainers.forEach((data, spine) => {
        const { container } = data;

        // 更新标签可见性
        const shouldBeVisible = appStore.showBoneNames && spine.parent !== null; // 只有当spine在舞台上时才显示标签
        if (container.visible !== shouldBeVisible) {
            container.visible = shouldBeVisible;
        }

        // 如果可见，则更新位置
        if (container.visible) {
            updateBoneLabelPosition(spine);
        }
    });
}

// 清理骨骼名称标签
function cleanupBoneLabels() {
    boneLabelsContainers.forEach((data, spine) => {
        if (data.container.parent) {
            data.container.parent.removeChild(data.container);
        }
        data.container.destroy({ children: true });
    });
    boneLabelsContainers.clear();
}

// 更新Spine模型，供App.vue调用
function updateSpineModels() {
    // 清理旧的标签
    cleanupBoneLabels();
    
    // 为当前活动容器中的所有Spine对象创建新标签
    const activeContainer = appStore.getActive();
    if (activeContainer && activeContainer.stage) {
        // 延迟执行，确保Spine模型已经完全加载
        setTimeout(() => {
            console.log("更新骨骼名称标签，当前显示状态:", appStore.showBoneNames);
            console.log("舞台子元素数量:", activeContainer.stage.children.length);
            
            activeContainer.stage.children.forEach(child => {
                if (child instanceof Spine) {
                    console.log("找到Spine对象:", child);
                    createBoneLabels(child);
                }
            });
            
            // 强制更新一次
            updateBoneLabels();
        }, 200); // 增加延迟时间，确保模型完全加载
    }
}

// 暴露方法给父组件
defineExpose({
    updateSpineModels
});

// 监听骨骼名称显示开关的变化
watch(() => appStore.showBoneNames, (show) => {
    console.log("骨骼名称显示开关变化:", show);
    
    boneLabelsContainers.forEach(data => {
        data.container.visible = show;
    });
    
    // 如果开启显示，强制更新一次位置
    if (show) {
        updateBoneLabels();
    }
});

// 监听活动容器的变化，更新骨骼名称标签
watch(() => appStore.activeIndex, () => {
    // 清理旧的标签
    cleanupBoneLabels();
    
    // 为当前活动容器中的所有Spine对象创建新标签
    const activeContainer = appStore.getActive();
    if (activeContainer && activeContainer.stage) {
        // 延迟执行，确保Spine模型已经完全加载
        setTimeout(() => {
            console.log("活动容器变化，更新骨骼名称标签");
            
            activeContainer.stage.children.forEach(child => {
                if (child instanceof Spine) {
                    createBoneLabels(child);
                }
            });
            
            // 强制更新一次
            updateBoneLabels();
        }, 200); // 增加延迟时间，确保模型完全加载
    }
});

// 在每一帧更新骨骼名称标签的位置
pixiApp.ticker.add(updateBoneLabels);

let resizeTimer, resizeTimer2;
const displayObserver = new ResizeObserver((entry) => {
    pixiApp.renderer.resize(entry[0].contentRect.width, entry[0].contentRect.height)
    clearTimeout(resizeTimer)
    clearTimeout(resizeTimer2)
    pixiApp.view.style.opacity = '0'
    data.resolution.show = true
    data.resolution.width = pixiApp.renderer.width
    data.resolution.height = pixiApp.renderer.height
    resizeTimer = setTimeout(() => {
        data.resolution.show = false
    }, 1500)
    resizeTimer2 = setTimeout(() => {
        pixiApp.view.style.opacity = '1'
    }, 100)
});

watch(() => appStore.active.container.data.position.x, x => {
    data.coordinate.x = x
}, {immediate: true})

watch(() => appStore.active.container.data.position.y, y => {
    data.coordinate.y = y
}, {immediate: true})

watch(() => appStore.active.container.spineVersion.value, version => {
    data.spineVersion = version
})

onMounted(() => {
    const display = getById('display')
    // pixiApp.resizeTo = display.value
    display.append(pixiApp.view)
    displayObserver.observe(display);
    appStore.getActive().data.position.x = pixiApp.view.clientWidth / 2
    appStore.getActive().data.position.y = pixiApp.view.clientHeight / 2

    display.addEventListener('contextmenu', (ev) => {
        ev.preventDefault()
        ipcRenderer.send('show-context-menu', {x: ev.x, y: ev.y})
    })
    ipcRenderer.on('copy-image', () => {
        pixiApp.view.toBlob(blob => {
            navigator.clipboard.write([new ClipboardItem({'image/png': blob})])
        })
    })
    
    // 初始化骨骼名称标签
    // 延迟执行，确保Spine模型已经完全加载
    setTimeout(() => {
        console.log("初始化骨骼名称标签");
        
        const activeContainer = appStore.getActive();
        if (activeContainer && activeContainer.stage) {
            activeContainer.stage.children.forEach(child => {
                if (child instanceof Spine) {
                    createBoneLabels(child);
                }
            });
            
            // 强制更新一次
            updateBoneLabels();
        }
    }, 200); // 增加延迟时间，确保模型完全加载
})

// 在组件销毁时清理资源
onUnmounted(() => {
    // 移除ticker
    if (pixiApp && pixiApp.ticker) {
        pixiApp.ticker.remove(updateBoneLabels);
    }
    
    // 清理骨骼名称标签
    cleanupBoneLabels();
});

pixiApp.view.addEventListener('wheel', (ev) => {
    ev.preventDefault();
    const minScale = 0.01, maxScale = 10;
    const active = appStore.getActive()
    if (ev.ctrlKey) {
        if (!active.background) return
        const originalScale = active.background.scale.x
        const scaleFactor = ev.deltaY > 0 ? 0.95 : 1.05;
        const newScale = Math.min(Math.max(originalScale * scaleFactor, minScale), maxScale)

        active.background.scale.set(newScale)
        active.background.x -= (ev.offsetX - active.background.x) * (newScale / originalScale - 1);
        active.background.y -= (ev.offsetY - active.background.y) * (newScale / originalScale - 1);
    } else {
        if (active.textures.length <= 0) return
        const originalScale = active.data.zoom
        const scaleFactor = ev.deltaY > 0 ? 0.95 : 1.05;
        const newScale = Math.min(Math.max(originalScale * scaleFactor, minScale), maxScale)

        active.data.zoom = newScale
        active.data.position.x -= (ev.offsetX - active.data.position.x) * (newScale / originalScale - 1);
        active.data.position.y -= (ev.offsetY - active.data.position.y) * (newScale / originalScale - 1);
    }
})

let isDragging = false;
let mouseX, mouseY, deltaX, deltaY;
pixiApp.view.addEventListener('pointerdown', (ev) => {
    if (appStore.getActive().textures.length === 0 && !appStore.getActive().background) return
    if (ev.button === 0) {
        isDragging = true;
        mouseX = ev.clientX;
        mouseY = ev.clientY;
    }
});
pixiApp.view.addEventListener('pointermove', (ev) => {
    if (isDragging) {
        deltaX = ev.clientX - mouseX;
        deltaY = ev.clientY - mouseY;

        const active = appStore.getActive()
        if (ev.ctrlKey) {
            if (!active.background) return;
            active.background.position.x += deltaX
            active.background.position.y += deltaY
        } else {
            if (active.textures.length === 0) return
            active.data.position.x += deltaX
            active.data.position.y += deltaY
        }

        mouseX = ev.clientX;
        mouseY = ev.clientY;
    }
});
pixiApp.view.addEventListener('pointerup', () => {
    isDragging = false;
});
pixiApp.view.addEventListener('pointerout', () => {
    isDragging = false;
});

</script>

<style scoped>
#display {
    flex-grow: 1;
    overflow: hidden;
    position: relative;
    height: 100% !important;
    background-repeat: no-repeat;
}

#display::before,
#display::after {
    z-index: -1;
    content: "";
    position: absolute;
    background-color: #c8c8c8;
}

#display::before {
    top: 50%;
    width: 100%;
    height: 1px;
}

#display::after {
    left: 50%;
    width: 1px;
    height: 100%;
}

#resolution {
    top: 5px;
    left: 10px;
    z-index: 8;
    color: #d2d2d2;
    position: absolute;
}

#coordinate {
    top: 5px;
    right: 10px;
    z-index: 8;
    color: #d2d2d2;
    position: absolute;
}

#spine-version {
    color: #d2d2d2;
    position: fixed;
    right: 5px;
    bottom: 5px;
}
</style>