<template>
  <view class="h-screen flex flex-col">
    <text class="text-center">主页 {{ new Date() }}</text>
    <view class="flex-1 overflow-hidden">
      <virtual-list :list="list" positionMode="absolute" ref="virtualListRef">
        <template #default="{ item, index }">
          <view
            :style="{ height: `${item.height}px` || '100rpx' }"
            :class="index % 2 === 0 ? 'bg-red-500' : 'bg-blue-500'"
            @click="handleClick(item, index)"
          >
            {{ item }}
          </view>
        </template>
      </virtual-list>
    </view>
  </view>
</template>

<script setup lang="ts">
import VirtualList from '@/components/virtualList/VirtualList.vue'

const list = Array.from({ length: 10000 }, (_, index) => ({ name: `item${index}`, id: index }))

const virtualListRef = ref()

const handleClick = (item: any, index: number) => {
  item.height = '120'

  nextTick(() => {
    virtualListRef.value.updateElementHeight(index)
  })
}
</script>
