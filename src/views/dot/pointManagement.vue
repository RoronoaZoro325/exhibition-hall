<script setup lang="ts">
import { ref, toRaw, reactive, onMounted } from 'vue'
import FloorMap from './floorMap.vue'

import floorImg1 from '../../assets/dot/1.png'
import floorImg2 from '../../assets/dot/2.jpg'
import floorImg3 from '../../assets/dot/3.jpg'
import floorImg4 from '../../assets/dot/4.jpeg'
import deviceImg1 from '../../assets/dot/icon1.png'
import deviceImg2 from '../../assets/dot/icon2.png'
import deviceImg3 from '../../assets/dot/icon3.png'
import deviceImg4 from '../../assets/dot/icon4.png'
import deviceImg5 from '../../assets/dot/icon5.png'
import deviceImg6 from '../../assets/dot/icon6.png'
import deviceImg7 from '../../assets/dot/icon7.png'
import deviceImg8 from '../../assets/dot/icon8.png'
import deviceImg9 from '../../assets/dot/icon9.png'

interface FloorChildren {
  id: number
  level: number
  label: string
  url: string
}

interface FloorObj {
  id: number
  level: number
  label: string
  children: FloorChildren[]
}

// interface DeviceChildren {
//   id: number
//   label: string
//   url: string
//   color: string
// }

interface Marker {
  dragFromFather: boolean
  id: number
  label: string
  left: number
  top: number
  url: string
}

const treeProps: object = {
  children: 'children',
  label: 'label'
}

const floorMapRef = <any>ref(null)
const deviceTree = ref<Marker[]>([
  {
    id: 0,
    label: '点位1',
    url: deviceImg1,
    dragFromFather: false,
    left: 0,
    top:0
  },
  {
    id: 1,
    label: '点位2',
    url: deviceImg2,
    dragFromFather: false,
    left: 0,
    top:0
  },
  {
    id: 2,
    label: '点位3',
    url: deviceImg3,
    dragFromFather: false,
    left: 0,
    top:0
  },
  {
    id: 3,
    label: '点位4',
    url: deviceImg4,
    dragFromFather: false,
    left: 0,
    top:0
  },
  {
    id: 4,
    label: '点位5',
    url: deviceImg5,
    dragFromFather: false,
    left: 0,
    top:0
  },
  {
    id: 5,
    label: '点位6',
    url: deviceImg6,
    dragFromFather: false,
    left: 0,
    top:0
  },
  {
    id: 6,
    label: '点位7',
    url: deviceImg7,
    dragFromFather: false,
    left: 0,
    top:0
  },
  {
    id: 7,
    label: '点位8',
    url: deviceImg8,
    dragFromFather: false,
    left: 0,
    top:0
  },
  {
    id: 8,
    label: '点位9',
    url: deviceImg9,
    dragFromFather: false,
    left: 0,
    top:0
  }
])
const markers = ref<Marker[]>([])
const currentFloorUrl = ref<string>('')
const dragIcon = ref('')
const floorTree = reactive<FloorObj[]>([
  {
    id: 1,
    level: 0,
    label: '访客中心',
    children: [
      {
        id: 1,
        level: 1,
        label: '一楼',
        url: floorImg1
      },
      {
        id: 2,
        level: 1,
        label: '二楼',
        url: floorImg2
      }
    ]
  },
  {
    id: 4,
    level: 0,
    label: '东食堂',
    children: [
      {
        id: 1,
        level: 1,
        label: '一楼',
        url: floorImg3
      },
      {
        id: 2,
        level: 1,
        label: '二楼',
        url: floorImg4
      }
    ]
  }
])

onMounted(() => {
  currentFloorUrl.value = floorTree[0].children[0].url
  let oldMarkers = JSON.parse(localStorage.getItem('markers') as string)
  markers.value = oldMarkers ? oldMarkers : []
  // 去除底图上已存在的点位信息
  deviceTree.value = deviceTree.value.filter((obj1) => {
    return !markers.value.some((obj2) => obj2.id === obj1.id)
  })
})

const saveMarkersPosition = () => {
  let oldMarkers = JSON.parse(localStorage.getItem('markers') as string)
  // 点位去重并保存
  let uniqueMarkers = oldMarkers ? new Map(oldMarkers.map((v: Marker) => [v.id, v])) : new Map()
  markers.value.map((v) => uniqueMarkers.set(v.id, v))
  markers.value = [...uniqueMarkers.values()] as Marker[]
  let temp = toRaw(markers.value)
  localStorage.setItem('markers', JSON.stringify(temp))
}

const changeMarkers = (newMarkers: Marker[]) => {
  markers.value = newMarkers
  saveMarkersPosition()
}

const deleteMarker = (marker: Marker) => {
  deviceTree.value.push(marker)
  deviceTree.value.sort((a, b) => a.id - b.id)
  markers.value = markers.value.filter((v) => v.id != marker.id)
  let temp = toRaw(markers.value)
  localStorage.setItem('markers', JSON.stringify(temp))
}

const getContinuousPoints = (id: number) => {
  deviceTree.value = deviceTree.value.filter((v) => v.id != id)
}

const nodeClick = (data: FloorObj & FloorChildren) => {
  console.log('🚀 ~ nodeClick ~ data:', data)
  currentFloorUrl.value = data.url || currentFloorUrl.value
  floorMapRef.value.mapInit()
}

const dragstart = (e: any, data: Marker) => {
  dragIcon.value = e.target.children[1]
  e.dataTransfer.setDragImage(dragIcon.value, 10, 10)
  data.dragFromFather = true // 告诉子组件拖拽事件的来源
  let temp = toRaw(data)
  e.dataTransfer.setData('text/plain', JSON.stringify(temp))
}

</script>

<template>
  <div class="point-management">
    <div class="floor-tree tree">
      <el-tree
        node-key="id"
        default-expand-all
        :highlight-current="true"
        :expand-on-click-node="false"
        :data="floorTree"
        @node-click="nodeClick"
        :props="treeProps"
      >
      </el-tree>
    </div>
    <floor-map
      @change-markers="changeMarkers"
      @get-continuous-points="getContinuousPoints"
      @delete-marker="deleteMarker"
      ref="floorMapRef"
      :currentFloorUrl="currentFloorUrl"
      :markers="markers"
      :continuousPoints="deviceTree"
    />
    <div class="device-tree tree">
      <el-tree
        ode-key="id"
        default-expand-all
        :highlight-current="true"
        :expand-on-click-node="false"
        :data="deviceTree"
        ><template #default="{ node, data }">
          <!-- https://developer.mozilla.org/zh-CN/docs/Web/API/DataTransfer/setDragImage -->
          <span :draggable="true" @dragstart="dragstart($event, data)">
            <span>{{ node.label }}{{ node.url }}</span>
            <span>
              <img class="drag-icon" :src="data.url" />
            </span>
          </span>
        </template>
      </el-tree>
    </div>
  </div>
</template>

<style lang="scss">
.point-management {
  height: 90vh;
  display: flex;
  justify-content: center;
  align-items: center;
  overflow: hidden;
  .tree {
    height: 90vh;
    position: fixed;
    background: white;
    z-index: 99;
    top: 10px;
    width: 250px;
    padding: 5px;
    box-shadow: 0 0 10px rgba(0, 0, 0, 0.5);
  }

  .floor-tree {
    left: 10px;
  }

  .device-tree {
    right: 10px;

    .drag-icon {
      width: 20px;
      margin-left: 5px;
    }
  }
}
</style>
