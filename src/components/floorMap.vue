<script setup lang="ts">
import { ref, reactive, onMounted, onUnmounted, getCurrentInstance, defineEmits, toRaw } from 'vue'
const { proxy }: any = getCurrentInstance()

interface Marker {
  color: string
  dragFromFather: boolean
  id: number
  label: string
  left: number
  top: number
  url: string
}

interface Member {
  left: number
  top: number
}

const props = defineProps<{
  currentFloorUrl: string
  markers: Marker[]
  continuousPoints: Marker[]
}>()

const emitEvents = defineEmits<{
  (e: 'changeMarkers', newMarkers: Marker[]): void
  (e: 'getContinuousPoints', id: number): void
  (e: 'deleteMarker', marker: Marker): void
}>()

const num = ref<number>(1.5) // 底图缩放等级
const draggingMarker = ref<Marker | null>(null) // 判断拖拽的是底图还是标记点，如果是后者，给予该标记点的信息
const continuous = ref<boolean>(false) //是否连续打点
const popoverInfo = ref<Marker | null>(null)
const popoverVisible = ref<boolean>(false)
const isDragging = ref<boolean>(false) // 是否正在拖拽标记点
const ST = ref<number>(0) // 弹窗延时器
const draggedMap = ref<boolean>(false) // 是否拖拽了底图
const zoomedMap = ref<boolean>(false) // 是否缩放了底图
const markerWidth = ref<number>(20) // 标记点宽度
const initPosition: Member = reactive({
  // 底图初始位置
  left: 0,
  top: 0
})

onMounted(() => {
  initPosition.left = proxy.$refs.floorMap.style.left
  initPosition.top = proxy.$refs.floorMap.style.top
})

const mapInit = () => {
  num.value = 1.5
  proxy.$refs.floorMap.style.left = initPosition.left
  proxy.$refs.floorMap.style.top = initPosition.top
}

//缩放底图
const zoomMap = (e: any) => {
  draggedMap.value = true
  popoverVisible.value = false
  if (e.deltaY > 0) {
    if (num.value < 0.5) return
    num.value -= 0.1
  } else {
    num.value += 0.1
  }
}

const markerStyle = (marker: Marker): any => {
  return {
    position: 'absolute',
    left: marker.left + 'px',
    top: marker.top + 'px',
    scale: 1 / num.value,
    width: markerWidth.value + 'px'
  }
}

const popoverStyle = (marker: Marker | null) => {
  return {
    left: marker?.left + 'px',
    top: marker?.top + 'px'
  }
}

const showPopover = (e: any, marker: Marker) => {
  clearTimeout(ST.value)
  if (isDragging.value) return
  if (!continuous.value) popoverVisible.value = true
  // 取鼠标进入的第一个坐标点，之后的鼠标划动事件不再监听
  if (marker.id !== popoverInfo.value?.id || draggedMap.value || zoomedMap.value) {
    draggedMap.value = false
    zoomedMap.value = false
    popoverInfo.value = Object.assign(JSON.parse(JSON.stringify(marker)), {
      left: e.clientX - e.offsetX + markerWidth.value / 2, // 10=标记点宽度/2
      top: e.clientY - e.offsetY,
      id: marker.id
    })
  }
}

const hidePopover = () => {
  if (isDragging.value) return
  ST.value = window.setTimeout(() => {
    popoverVisible.value = false
  }, 1000)
}

const slideIntoMark = () => {
  clearTimeout(ST.value)
}

const slideOutMark = () => {
  popoverVisible.value = false
}

// 点击底图或者标记点，以及后续的拖动
const clickMap = (e: any) => {
  let odiv = e.target
  let scale = draggingMarker.value ? num.value : 1 // draggingMarker 判断拖拽的是底图还是标记点，后者的话加上num当系数
  zoomedMap.value = !draggingMarker.value
  popoverVisible.value = false
  // 计算鼠标距离底图左边界的初始距离
  let disX = e.clientX - odiv.offsetLeft * scale
  let disY = e.clientY - odiv.offsetTop * scale
  document.onmousemove = (e) => {
    let left = e.clientX - disX
    let top = e.clientY - disY
    odiv.style.left = left / scale + 'px'
    odiv.style.top = top / scale + 'px'
  }
  document.onmouseup = (e: any) => {
    if (draggingMarker.value) {
      let left = e.clientX - disX
      let top = e.clientY - disY
      let markers = JSON.parse(JSON.stringify(toRaw(props.markers)))
      markers.map((v: Marker) => {
        if (v.id == draggingMarker.value?.id) {
          v.top = top / scale
          v.left = left / scale
        }
      })
      emitEvents('changeMarkers', markers)
      draggingMarker.value = null
    }
    document.onmousemove = null
    document.onmouseup = null
    isDragging.value = false
  }
}

// 设置连续打点时的鼠标位置
const setCustomizeMouse = (e: any) => {
  proxy.$refs.customizeMouseShape.style.left = e.clientX - 10 + 'px'
  proxy.$refs.customizeMouseShape.style.top = e.clientY - 10 + 'px'
}

// 结束连续打点
const endContinuous = () => {
  continuous.value = false
  document.onclick = null
  proxy.$refs.floorMap.onmousemove = null
}

// 拖拽过来之后设置点位
const setMarkerPosition = (e: any, odiv: any, dragData: Marker) => {
  let markers = JSON.parse(JSON.stringify(toRaw(props.markers)))
  markers.push(
    Object.assign(dragData, {
      top: (e.clientY - odiv.getBoundingClientRect().y) / num.value - markerWidth.value / 2,
      left: (e.clientX - odiv.getBoundingClientRect().x) / num.value - markerWidth.value / 2
    })
  )
  emitEvents('getContinuousPoints', dragData.id)
  emitEvents('changeMarkers', markers)
}

// 拖拽结束事件
const ondrop = (e: any) => {
  if (!e.dataTransfer.getData('text/plain')) return // 是否携带数据
  let dragData = JSON.parse(e.dataTransfer.getData('text/plain'))
  if (!dragData.dragFromFather) return // 是否来自父组件的拖拽
  let odiv = e.target
  setMarkerPosition(e, odiv, dragData)
  if (props.continuousPoints.length === 1) return // 如果只有一个就不用开启连续打点
  continuous.value = true
  proxy.$refs.floorMap.onmousemove = (e: any) => {
    setCustomizeMouse(e)
  }

  // 鼠标左键点击
  document.onclick = (e: any) => {
    if (props.continuousPoints.length === 1) {
      endContinuous()
    }
    let dragData = props.continuousPoints[0]
    setMarkerPosition(e, odiv, dragData)
  }
}

const markerClick = (item: Marker) => {
  draggingMarker.value = item
  popoverVisible.value = false
  popoverInfo.value = null
  isDragging.value = true
}

const deleteMarker = () => {
  emitEvents('deleteMarker', popoverInfo.value as Marker)
  popoverInfo.value = null
  popoverVisible.value = false
}

// 鼠标右键事件
const oncontextmenu = (e: any) => {
  e.preventDefault()
  endContinuous()
}

onUnmounted(() => {
  document.onmousemove = null
  document.onmouseup = null
  document.onclick = null
})

defineExpose({
  mapInit
})
</script>

<template>
  <!-- 必须给拖放区元素添加 dragover.prevent，才能使 drop 事件正确执行 -->
  <div class="container" @dragover.prevent @drop="ondrop" @contextmenu="oncontextmenu">
    <div
      ref="floorMap"
      class="floorMap"
      :style="{
        transform: `scale(${num})`,
        cursor: continuous ? 'none' : 'default'
      }"
      @mousewheel="zoomMap"
      @mousedown="clickMap"
    >
      <!-- click在鼠标释放时触发，所以要用mousedown，在按下的时候就赋值 -->
      <img
        v-for="(marker, index) in markers"
        @mousedown="markerClick(marker)"
        @mouseenter="showPopover($event, marker)"
        @mouseleave="hidePopover()"
        :style="markerStyle(marker)"
        :key="index"
        class="marker-class"
        :src="marker.url"
      />
      <img class="base-map" :src="currentFloorUrl" />
    </div>
    <div
      v-if="popoverVisible"
      class="popover"
      :style="popoverStyle(popoverInfo)"
      @mouseenter="slideIntoMark"
      @mouseleave="slideOutMark"
    >
      <div class="content">
        <span>点位{{ popoverInfo?.id }}</span>
        <span @click="deleteMarker" class="delete">删除</span>
      </div>
      <div class="popover-arrow"></div>
    </div>
    <img
      v-if="continuous"
      class="customize-mouse-shape"
      ref="customizeMouseShape"
      src="../assets/dot.png"
    />
  </div>
</template>

<style lang="scss" scoped>
.container {
  width: 1200px;
  height: 900px;
  margin: 0 auto;
  position: relative;

  .floorMap {
    position: relative;
    width: 100%;
    height: 100%;

    .marker-class {
      -webkit-user-drag: none;
      -moz-user-drag: none;
      -ms-user-drag: none;
      user-drag: none;
    }

    .base-map {
      width: 100%;
      height: 100%;
      pointer-events: none;
      user-select: none;
    }
  }

  .customize-mouse-shape {
    position: fixed;
    width: 20px;
    // 虽然鼠标形状变了，但不是cursor设置的自定义图片，本质是dom，要禁止拖拽和选中，否则容易出现误操作
    user-select: none;
    -webkit-user-drag: none;
    -moz-user-drag: none;
    -ms-user-drag: none;
    user-drag: none;
    pointer-events: none;
  }

  .popover {
    position: fixed;
    font-size: 18px;
    color: white;
    transform: translate(-50%, calc(-100% - 10px));
    width: 200px;
    background: rgba(0, 0, 0, 0.7);
    padding: 15px;
    border-radius: 5px;
    user-select: none;

    .content {
      display: flex;
      justify-content: space-between;

      .delete {
        background: red;
        padding: 5px 10px;
        cursor: pointer;
        font-size: 14px;
        border-radius: 3px;
        user-select: none;
      }
    }

    .popover-arrow {
      position: absolute;
      width: 0;
      height: 0;
      border-style: solid;
      border-width: 8px;
      border-color: rgba(0, 0, 0, 0.7) transparent transparent transparent;
      transform: translateY(calc(100% - 1px));
      left: calc(50% + 4px); // 4=箭头宽度8/2
      margin-left: -10px;
    }
  }
}
</style>
