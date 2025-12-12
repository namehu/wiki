<template lang="pug">
//- 容器继承父级的背景色 color，并使用 flex 布局占满高度
.d-flex.flex-column.fill-height(:class='color')
  //- 1. 静态头部区域 (面包屑 + 当前页信息)
  v-list.flex-grow-0.pb-0(dense, :class='color', :dark='dark', style="background: transparent;")
    template(v-if='currentParent.id > 0')
      v-list-item(
        v-for='(item, idx) of parents',
        :key='`parent-` + item.id',
        @click='$emit("fetch-items", item)',
        style='min-height: 30px;'
      )
        v-list-item-avatar(size='18', :style='`padding-left: ` + (idx * 8) + `px; width: auto; margin: 0 5px 0 0;`')
          v-icon(small) mdi-folder-open
        v-list-item-title {{ item.title }}

      v-divider.mt-2

      v-list-item.mt-2(
        v-if='currentParent.pageId > 0',
        :href='`/` + currentParent.locale + `/` + currentParent.path',
        :key='`directorypage-` + currentParent.id',
        :input-value='path === currentParent.path'
      )
        v-list-item-avatar(size='24')
          v-icon mdi-text-box
        v-list-item-title {{ currentParent.title }}

      v-subheader.pl-4.height-auto.py-2(style="height: 30px;") {{$t('common:sidebar.currentDirectory')}}

  //- 2. 搜索框 (融合样式)
  //- 增加 v-if 判断：当 currentParent.pageId > 0 时不渲染此节点
  .px-3.pb-2.pt-1(v-if="!(currentParent.id > 0)")
    v-text-field(
      v-model='searchQuery'
      :label='$t("action.search")'
      prepend-inner-icon='mdi-magnify'
      flat
      solo-inverted
      hide-details
      dense
      :dark='dark'
      clearable
      style="font-size: 14px;"
    )

  //- 3. 虚拟列表容器 (flex-grow-1 自动填满剩余空间)
  //- 使用 resizeObserver 动态计算高度，确保数据不会显示不全
  .flex-grow-1.virtual-scroll-wrapper(ref='scrollContainer', style="position: relative; overflow: hidden;")
    v-virtual-scroll(
      v-if='containerHeight > 0'
      :items='filteredItems'
      :height='containerHeight'
      :item-height='40'
      :bench='5'
      style="overflow-y: auto;"
    )
      template(v-slot:default='{ item }')
        //- 文件夹渲染
        v-list-item(
          v-if='item.isFolder',
          :key='`childfolder-` + item.id',
          @click='$emit("fetch-items", item)',
          dense,
          :dark='dark'
          style="height: 40px;"
          color="white"
        )
          v-list-item-avatar(size='24')
            v-icon mdi-folder
          v-list-item-title {{ item.title }}

        //- 页面渲染
        v-list-item(
          v-else,
          :href='`/` + item.locale + `/` + item.path',
          :key='`childpage-` + item.id',
          :input-value='path === item.path',
          dense,
          :dark='dark'
          style="height: 40px;"
          color="white"
        )
          v-list-item-avatar(size='24')
            v-icon mdi-text-box
          v-list-item-title {{ item.title }}

</template>

<script>
import _ from 'lodash'

export default {
  name: 'NavSidebarBrowse',
  props: {
    items: { type: Array, default: () => [] },
    parents: { type: Array, default: () => [] },
    currentParent: { type: Object, default: () => ({}) },
    path: { type: String, default: '' },
    locale: { type: String, default: 'en' },
    color: { type: String, default: 'primary' },
    dark: { type: Boolean, default: true }
  },
  data() {
    return {
      searchQuery: '',
      containerHeight: 0,
      resizeObserver: null
    }
  },
  computed: {
    filteredItems() {
      if (!this.searchQuery) return this.items
      const lowerQuery = this.searchQuery.toLowerCase()
      return this.items.filter(item => {
        return (item.title && item.title.toLowerCase().includes(lowerQuery)) ||
               (item.path && item.path.toLowerCase().includes(lowerQuery))
      })
    }
  },
  mounted() {
    this.initResizeObserver()
  },
  beforeDestroy() {
    if (this.resizeObserver) {
      this.resizeObserver.disconnect()
    }
  },
  methods: {
    initResizeObserver() {
      const el = this.$refs.scrollContainer
      if (!el) return

      // 监听容器高度变化，动态赋值给 virtual-scroll
      this.resizeObserver = new ResizeObserver(_.throttle(entries => {
        for (let entry of entries) {
          this.containerHeight = entry.contentRect.height
        }
      }, 50))

      this.resizeObserver.observe(el)
    }
  },
  watch: {
    currentParent() {
      // 切换目录时清空搜索
      this.searchQuery = ''
      // 切换目录后可能列表变了，或者是搜索框消失/出现导致高度变化，强制触发一次高度重算
      this.$nextTick(() => {
        if (this.$refs.scrollContainer) {
          this.containerHeight = this.$refs.scrollContainer.clientHeight
        }
      })
    }
  }
}
</script>

<style scoped>
/* 强制覆盖 v-virtual-scroll 内部可能出现的样式问题 */
.virtual-scroll-wrapper >>> .v-virtual-scroll {
  /* 隐藏原生的滚动条轨道，根据需求可选 */
}
/* 确保列表项内容垂直居中 */
.v-list-item {
  min-height: 40px;
}
</style>
