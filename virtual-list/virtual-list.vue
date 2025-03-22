<template>
  <scroll-view
    class="cache-container"
    scroll-y
    upper-threshold="100"
    lower-threshold="100"
    @scrolltolower="handleScrollToLower"
    @scrolltoupper="handleScrollToUpper"
    @scroll="handleScroll"
    :scroll-top="scrollTop"
  >
    <!-- 使用缓冲区容器 -->
    <view
      v-for="(item, index) in virtualList"
      :key="`virtual-${item.index}-${index}`"
      :id="`virtual-${item.index}`"
      class="absolute top-0 w-full virtual-item"
      :style="{
        transform: `translate3d(0, ${item.translateY}px, 0)`,
        zIndex: item.index,
      }"
      :data-index="item.index"
    >
      <slot :item="item" :index="index" :list="virtualList"></slot>
    </view>
  </scroll-view>
</template>

<script setup lang="ts">
import { nextTick, onMounted, ref, watch, getCurrentInstance } from 'vue'

const props = defineProps({
  list: {
    type: Array<Object>,
    default: () => [],
  },
  virtualListLength: {
    type: Number,
    default: 20,
  },
  itemHeight: {
    type: Number,
    default: 100,
  },
  // 新增预加载数量参数，提前加载上下元素避免白屏
  preloadCount: {
    type: Number,
    default: 10,
  },
  // 开启动态高度测量
  dynamicHeight: {
    type: Boolean,
    default: true,
  },
  // 是否在滚动时持续更新高度
  continuousHeight: {
    type: Boolean,
    default: true,
  },
  // 高度样本数量
  sampleCount: {
    type: Number,
    default: 5,
  },
  // 高度测量延迟
  measureDelay: {
    type: Number,
    default: 100,
  },
  // 位置计算模式：absolute（绝对定位）或cumulative（累计定位）
  positionMode: {
    type: String,
    default: 'cumulative',
    validator: (value: 'absolute' | 'cumulative') => ['absolute', 'cumulative'].includes(value),
  },
})

const virtualList = ref([]) // 渲染的虚拟滚动列表
const copyList = ref([]) // 复制的列表
const scrollTop = ref(0) // 虚拟滚动的时候记录滚动位置
const totalHeight = ref(0) // 整个列表的总高度
const itemHeights = ref([]) // 存储每个项目的实际高度
const averageHeight = ref(props.itemHeight) // 计算得到的平均高度
const measuredIndices = ref(new Set()) // 已测量过高度的元素索引
const currentIndex = ref(0)
const isCalculating = ref(false) // 防止快速滑动时重复计算
const lastMeasureTime = ref(0) // 最后一次测量时间
const positionCache = ref({}) // 位置缓存，优化计算
const accumulatedHeights = ref([]) // 累积高度数组
const isScrolling = ref(false) // 是否正在滚动
const scrollTimeout = ref(null) // 滚动状态超时定时器
const heightUpdateTimeout = ref(null) // 高度更新定时器
const instance = getCurrentInstance()

// 使用节流函数优化滚动处理
const throttle = (fn, delay) => {
  let lastCall = 0
  return function (...args) {
    const now = new Date().getTime()
    if (now - lastCall < delay) return
    lastCall = now
    return fn(...args)
  }
}

// 获取滚动信息，使用节流优化
const handleScrollThrottled = throttle((scrollTop) => {
  // 标记正在滚动
  isScrolling.value = true

  // 清除之前的滚动定时器
  if (scrollTimeout.value) {
    clearTimeout(scrollTimeout.value)
  }

  // 设置新的滚动超时，滚动停止后重新测量高度
  scrollTimeout.value = setTimeout(() => {
    isScrolling.value = false
    // 滚动停止后延迟测量，避免频繁计算
    if (props.continuousHeight) {
      scheduleHeightMeasurement(false)
    }
  }, 200)

  if (!isCalculating.value) {
    isCalculating.value = true
    // 微信小程序环境使用setTimeout代替requestAnimationFrame
    setTimeout(() => {
      updateVisibleItems(scrollTop)
      isCalculating.value = false
    }, 16) // 约60fps的更新频率
  }
}, 32) // 节流间隔

// 滚动处理函数
const handleScroll = (e) => {
  handleScrollThrottled(e.detail.scrollTop)
}

// 根据滚动位置更新可视区域的列表项
const updateVisibleItems = (scrollTop) => {
  if (copyList.value.length === 0) return

  // 估算当前应该显示的索引范围
  let estimatedIndex = getIndexByScrollTop(scrollTop)

  // 确保不会超出范围
  estimatedIndex = Math.max(
    0,
    Math.min(estimatedIndex, copyList.value.length - props.virtualListLength),
  )

  // 只有当索引变化较大时才重新计算，避免频繁更新
  if (Math.abs(estimatedIndex - currentIndex.value) > Math.floor(props.preloadCount / 3)) {
    currentIndex.value = estimatedIndex
    calculateVirtualList(
      Math.max(0, estimatedIndex - props.preloadCount),
      Math.min(
        copyList.value.length,
        estimatedIndex + props.virtualListLength + props.preloadCount,
      ),
    )
  }
}

// 根据滚动位置计算对应的索引 - 使用二分查找优化
const getIndexByScrollTop = (scrollTop) => {
  if (!accumulatedHeights.value.length) {
    return Math.floor(scrollTop / averageHeight.value)
  }

  // 使用二分查找定位索引 - O(log n)复杂度
  let low = 0
  let high = accumulatedHeights.value.length - 1

  while (low <= high) {
    const mid = Math.floor((low + high) / 2)
    if (accumulatedHeights.value[mid] < scrollTop) {
      low = mid + 1
    } else if (accumulatedHeights.value[mid] > scrollTop) {
      high = mid - 1
    } else {
      return mid
    }
  }

  return Math.max(0, low)
}

// 初始化项目高度数组
const initItemHeights = () => {
  // 初始化高度数组
  itemHeights.value = Array(copyList.value.length).fill(props.itemHeight)

  // 初始化位置缓存
  positionCache.value = {}

  // 计算累积高度
  updateAccumulatedHeights()

  // 延迟测量前几个元素作为样本，用于校准整体高度
  nextTick(() => {
    scheduleHeightMeasurement(true)
  })
}

// 计划高度测量，避免频繁测量
const scheduleHeightMeasurement = (isInitial = false) => {
  // 如果正在滚动且不是初始化测量，取消测量
  if (isScrolling.value && !isInitial) return

  // 清除现有的定时器
  if (heightUpdateTimeout.value) {
    clearTimeout(heightUpdateTimeout.value)
  }

  heightUpdateTimeout.value = setTimeout(() => {
    if (isInitial) {
      measureInitialSample()
    } else {
      measureItemHeights()
    }
    lastMeasureTime.value = Date.now()
    heightUpdateTimeout.value = null
  }, props.measureDelay)
}

// 更新累积高度数组
const updateAccumulatedHeights = () => {
  const heights = []
  let sum = 0

  for (let i = 0; i < itemHeights.value.length; i++) {
    sum += itemHeights.value[i] || averageHeight.value
    heights.push(sum)
  }

  accumulatedHeights.value = heights
  totalHeight.value = sum
}

// 测量初始样本以获取更准确的高度估计
const measureInitialSample = () => {
  // 只有当数据量足够大且启用动态高度时才需要样本测量
  if (!props.dynamicHeight || !virtualList.value.length) return

  //   console.log('开始测量初始样本高度')

  // 延迟执行测量，确保元素完全渲染
  setTimeout(() => {
    // 创建查询
    const query = uni.createSelectorQuery().in(instance.proxy)

    // 一次性查询所有可见元素
    query
      .selectAll('.virtual-item')
      .boundingClientRect((rects: any) => {
        if (!rects || !rects.length) {
          //   console.log('未找到元素，使用默认高度')
          return
        }

        let measuredCount = 0
        let totalHeight = 0

        // 处理查询结果
        rects.forEach((rect) => {
          if (rect && rect.dataset && rect.dataset.index !== undefined && rect.height) {
            const index = parseInt(rect.dataset.index)
            const height = Math.max(1, rect.height)

            // console.log('测量元素高度:', index, height)
            measuredCount++
            totalHeight += height

            // 记录已测量项
            measuredIndices.value.add(index)
            itemHeights.value[index] = height

            // 更新对应virtualList项的itemHeight
            const item = virtualList.value.find((item) => item.index === index)
            if (item) {
              item.itemHeight = height
            }

            // 清除该索引的位置缓存
            delete positionCache.value[index]
          }
        })

        if (measuredCount > 0) {
          // 如果有有效测量，更新平均高度
          const newAvgHeight = totalHeight / measuredCount
          averageHeight.value = newAvgHeight

          // 更新未测量元素的估计高度
          if (Math.abs(newAvgHeight - props.itemHeight) > props.itemHeight * 0.1) {
            for (let i = 0; i < itemHeights.value.length; i++) {
              if (!measuredIndices.value.has(i)) {
                itemHeights.value[i] = newAvgHeight
                delete positionCache.value[i]

                // 更新对应virtualList项的itemHeight估计值
                const item = virtualList.value.find((item) => item.index === i)
                if (item) {
                  item.itemHeight = newAvgHeight
                }
              }
            }
          }

          // 更新位置
          updateAccumulatedHeights()
          updateItemPositions()
        }
      })
      .exec()
  }, 200)
}

// 获取并更新每个项目的实际高度
const measureItemHeights = () => {
  if (!virtualList.value.length || !props.dynamicHeight) return

  //   console.log('测量列表项高度')

  // 延迟测量，确保元素已渲染
  setTimeout(() => {
    const query = uni.createSelectorQuery().in(instance.proxy)

    // 一次性查询所有可见元素
    query
      .selectAll('.virtual-item')
      .boundingClientRect((rects: any) => {
        if (!rects || !rects.length) return

        let measuredCount = 0
        let totalHeight = 0
        let heightsChanged = false

        // 处理查询结果
        rects.forEach((rect) => {
          if (rect && rect.dataset && rect.dataset.index !== undefined && rect.height) {
            const index = parseInt(rect.dataset.index)
            const height = Math.max(1, rect.height)

            measuredCount++
            totalHeight += height

            // 记录已测量项
            measuredIndices.value.add(index)

            // 只在高度有明显变化时更新
            if (!itemHeights.value[index] || Math.abs(itemHeights.value[index] - height) > 1) {
              itemHeights.value[index] = height

              // 更新对应virtualList项的itemHeight
              const item = virtualList.value.find((item) => item.index === index)
              if (item) {
                item.itemHeight = height
              }

              delete positionCache.value[index] // 清除缓存
              heightsChanged = true
            }
          }
        })

        // 如果有高度变化，重新计算位置
        if (heightsChanged && measuredCount > 0) {
          const newAverage = totalHeight / measuredCount
          if (!isNaN(newAverage) && newAverage > 0) {
            // 平滑过渡到新的平均高度
            averageHeight.value = averageHeight.value * 0.7 + newAverage * 0.3
          }

          // 更新位置
          updateAccumulatedHeights()
          updateItemPositions()
        }
      })
      .exec()
  }, 100)
}

// 更新列表项的位置信息
const calculateVirtualList = (startIndex, endIndex) => {
  //   console.log('计算虚拟列表:', startIndex, endIndex)

  // 防止越界
  startIndex = Math.max(0, startIndex)
  endIndex = Math.min(copyList.value.length, endIndex)

  // 保存旧的列表
  const oldList = [...virtualList.value]
  const oldIndices = oldList.map((item) => item.index)

  // 更新虚拟列表
  virtualList.value = copyList.value.slice(startIndex, endIndex).map((item, idx) => {
    // 保留之前已经计算好的translateY值
    let translateY = 0
    const existingItem = oldList.find((old) => old.index === item.index)

    if (existingItem && existingItem.translateY !== undefined) {
      translateY = existingItem.translateY
    }

    return {
      ...item,
      virtualIndex: idx,
      translateY,
      // 添加元素高度信息，如已测量则使用实际高度，否则使用平均高度
      itemHeight: itemHeights.value[item.index] || averageHeight.value,
    }
  })

  // 计算位置
  updateItemPositions()

  // 检查是否有新元素需要测量高度
  const hasNewItems = virtualList.value.some((item) => !oldIndices.includes(item.index))
  if (hasNewItems && props.dynamicHeight && !isScrolling.value) {
    scheduleHeightMeasurement()
  }
}

// 更新所有可见项的位置 - 重点优化此函数修复定位问题
const updateItemPositions = () => {
  if (props.positionMode === 'absolute') {
    // 绝对定位模式：每个元素都基于容器顶部定位
    updateItemPositionsAbsolute()
  } else {
    // 累计定位模式：基于前一个元素的位置计算（默认）
    updateItemPositionsCumulative()
  }
}

// 绝对定位模式：使用预计算的累积高度
const updateItemPositionsAbsolute = () => {
  virtualList.value.forEach((item) => {
    if (item.index === 0) {
      item.translateY = 0
      return
    }

    // 从位置缓存获取或计算新位置
    if (positionCache.value[item.index] !== undefined) {
      item.translateY = positionCache.value[item.index]
    } else {
      // 使用累积高度数组查找前一个元素的位置
      const position = accumulatedHeights.value[item.index - 1] || 0
      item.translateY = position
      positionCache.value[item.index] = position
    }
  })
}

// 累计定位模式：基于相邻元素位置计算
const updateItemPositionsCumulative = () => {
  // 按索引排序，确保从上到下依次计算
  const sortedList = [...virtualList.value].sort((a, b) => a.index - b.index)

  sortedList.forEach((item) => {
    if (item.index === 0) {
      item.translateY = 0
      return
    }

    // 从缓存获取
    if (positionCache.value[item.index] !== undefined) {
      item.translateY = positionCache.value[item.index]
      return
    }

    // 找到前一个相邻元素
    const prevIndex = item.index - 1
    const prevItem = sortedList.find((it) => it.index === prevIndex)

    if (prevItem && prevItem.translateY !== undefined) {
      // 基于前一个元素的位置 + 高度
      const prevHeight = itemHeights.value[prevIndex] || averageHeight.value
      item.translateY = prevItem.translateY + prevHeight
    } else {
      // 找不到前一个元素时使用累积高度
      item.translateY = accumulatedHeights.value[prevIndex] || 0
    }

    // 缓存计算结果
    positionCache.value[item.index] = item.translateY
  })
}

// 滚动到底部触发
const handleScrollToLower = () => {
  const newIndex = Math.min(
    copyList.value.length - props.virtualListLength,
    currentIndex.value + Math.floor(props.virtualListLength / 3),
  )

  if (newIndex > currentIndex.value) {
    currentIndex.value = newIndex
    calculateVirtualList(
      Math.max(0, newIndex - props.preloadCount),
      Math.min(copyList.value.length, newIndex + props.virtualListLength + props.preloadCount),
    )
  }
}

// 滚动到顶部触发
const handleScrollToUpper = () => {
  const newIndex = Math.max(0, currentIndex.value - Math.floor(props.virtualListLength / 3))

  if (newIndex < currentIndex.value) {
    currentIndex.value = newIndex
    calculateVirtualList(
      Math.max(0, newIndex - props.preloadCount),
      Math.min(copyList.value.length, newIndex + props.virtualListLength + props.preloadCount),
    )
  }
}

// 监听数据变化
watch(
  () => props.list,
  (newVal) => {
    if (!newVal || newVal.length === 0) {
      virtualList.value = []
      copyList.value = []
      totalHeight.value = 0
      measuredIndices.value.clear()
      positionCache.value = {}
      accumulatedHeights.value = []
      averageHeight.value = props.itemHeight
      return
    }

    // console.log('数据变化，重新初始化列表')

    // 初始化数据
    copyList.value = newVal.map((item: any, index) => ({
      ...item,
      index,
      id: item.id || `item-${index}`,
    }))

    // 重置测量状态
    measuredIndices.value.clear()
    positionCache.value = {}
    averageHeight.value = props.itemHeight

    // 初始化高度数组和位置
    initItemHeights()

    // 初始化虚拟列表
    calculateVirtualList(0, Math.min(newVal.length, props.virtualListLength + props.preloadCount))
  },
  {
    immediate: true,
    deep: true,
  },
)

// 在组件挂载后执行一次全面的高度测量
onMounted(() => {
  // 延迟执行，确保元素已渲染
  setTimeout(() => {
    scheduleHeightMeasurement(true)
  }, 300)
})

// 提供一个对外暴露的方法，允许手动刷新高度计算
const refreshHeight = () => {
  //   console.log('手动刷新高度计算')
  measuredIndices.value.clear()
  positionCache.value = {}
  scheduleHeightMeasurement(true)
}

// 更新特定元素高度并重新计算后续元素位置的方法
const updateElementHeight = (index: number) => {
  if (index < 0 || index >= copyList.value.length) {
    console.error('索引超出范围:', index)
    return
  }

  // 查找此索引对应的虚拟列表中的元素
  const virtualIndex = virtualList.value.findIndex((item) => item.index === index)
  if (virtualIndex === -1) {
    console.warn('当前元素不在可视区域内:', index)
    return
  }

  // 使用nextTick确保DOM已更新
  nextTick(() => {
    // 延迟执行测量
    setTimeout(() => {
      const query = uni.createSelectorQuery().in(instance.proxy)

      // 查询所有虚拟列表元素
      query
        .selectAll('.virtual-item')
        .boundingClientRect((rects: any) => {
          if (!rects || !rects.length) {
            console.warn('未找到任何虚拟列表元素')
            return
          }

          console.log('找到元素数量:', rects.length)

          // 从查询结果中找到匹配目标index的元素
          const targetRect = rects.find(
            (rect) => rect && rect.dataset && String(rect.dataset.index) === String(index),
          )

          if (!targetRect || !targetRect.height) {
            console.warn('未找到目标元素或高度无效:', index)
            return
          }

          const height = Math.max(1, targetRect.height)
          const oldHeight = itemHeights.value[index] || averageHeight.value

          // 只有高度发生明显变化时才更新
          if (Math.abs(oldHeight - height) <= 1) {
            return
          }

          console.log(`元素 ${index} 高度从 ${oldHeight}px 变为 ${height}px`)

          // 更新高度数组
          itemHeights.value[index] = height

          // 更新对应virtualList项的itemHeight
          const item = virtualList.value.find((item) => item.index === index)
          if (item) {
            item.itemHeight = height
          }

          // 更新copyList中的高度信息，以便在重新渲染时保留
          const originalItem = copyList.value.find((item) => item.index === index)
          if (originalItem) {
            originalItem.measuredHeight = height
          }

          // 清除此元素及之后所有元素的位置缓存
          for (let i = index; i < copyList.value.length; i++) {
            delete positionCache.value[i]
          }

          // 更新累积高度数组
          updateAccumulatedHeights()

          // 更新虚拟列表中受影响元素的位置
          updateItemPositions()

          // 触发一个通知，告知高度已更新
          uni.$emit('virtualListHeightChanged', { index, height, oldHeight })
        })
        .exec()
    }, 100) // 增加延迟时间以确保渲染完成
  })
}

// 对外暴露的方法
defineExpose({
  refreshHeight,
  updateElementHeight,
})
</script>

<style scoped>
.total-height {
  position: relative;
  pointer-events: none;
  height: 100%;
}
.virtual-item {
  width: 100%;
  box-sizing: border-box;
  will-change: transform; /* 优化渲染性能 */
}
.cache-container {
  position: relative;
  width: 100%;
  height: 100%;
  transform: translateZ(0); /* 启用硬件加速 */
  backface-visibility: hidden; /* 优化渲染 */
  perspective: 1000px;
  contain: layout paint style; /* 优化渲染性能 */
}
</style>

