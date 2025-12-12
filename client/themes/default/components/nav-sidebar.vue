<template lang="pug">
  div.d-flex.flex-column(style="height: calc(100vh - 64px); overflow: hidden;")
    //- 顶部 Tab 切换区
    .pa-3.d-flex.flex-grow-0(v-if='navMode === `MIXED`', :class='$vuetify.theme.dark ? `grey darken-5` : `blue darken-3`')
      v-btn(
        depressed
        :color='$vuetify.theme.dark ? `grey darken-4` : `blue darken-2`'
        style='min-width:0;'
        @click='goHome'
        :aria-label='$t(`common:header.home`)'
        )
        v-icon(size='20') mdi-home
      v-btn.ml-3(
        v-if='currentMode === `custom`'
        depressed
        :color='$vuetify.theme.dark ? `grey darken-4` : `blue darken-2`'
        style='flex: 1 1 100%;'
        @click='switchMode(`browse`)'
        )
        v-icon(left) mdi-file-tree
        .body-2.text-none {{$t('common:sidebar.browse')}}
      v-btn.ml-3(
        v-else-if='currentMode === `browse`'
        depressed
        :color='$vuetify.theme.dark ? `grey darken-4` : `blue darken-2`'
        style='flex: 1 1 100%;'
        @click='switchMode(`custom`)'
        )
        v-icon(left) mdi-navigation
        .body-2.text-none {{$t('common:sidebar.mainMenu')}}
    v-divider

    //-> Custom Navigation (普通菜单，保留 flex-grow-1 以便滚动)
    v-list.py-2.flex-grow-1(
      v-if='currentMode === `custom`'
      dense,
      :class='color',
      :dark='dark'
      style="overflow-y: auto;"
    )
      template(v-for='item of items')
        v-list-item(
          v-if='item.k === `link`'
          :href='item.t'
          :target='item.y === `externalblank` ? `_blank` : `_self`'
          :rel='item.y === `externalblank` ? `noopener` : ``'
          )
          v-list-item-avatar(size='24', tile)
            v-icon(v-if='item.c.match(/fa[a-z] fa-/)', size='19') {{ item.c }}
            v-icon(v-else) {{ item.c }}
          v-list-item-title {{ item.l }}
        v-divider.my-2(v-else-if='item.k === `divider`')
        v-subheader.pl-4(v-else-if='item.k === `header`') {{ item.l }}

    //-> Browse (使用新组件)
    //- 这里的 flex-grow-1 非常关键，它让 browse 组件占满剩余所有高度
    nav-sidebar-browse.flex-grow-1(
      v-else-if='currentMode === `browse`'
      :items="currentItems"
      :parents="parents"
      :current-parent="currentParent"
      :path="path"
      :locale="locale"
      :color="color"
      :dark="dark"
      @fetch-items="fetchBrowseItems"
    )
</template>

<script>
import _ from 'lodash'
import gql from 'graphql-tag'
import { get } from 'vuex-pathify'
import NavSidebarBrowse from './nav-sidebar-browse.vue'

/* global siteLangs */

export default {
  components: {
    NavSidebarBrowse
  },
  props: {
    color: {
      type: String,
      default: 'primary'
    },
    dark: {
      type: Boolean,
      default: true
    },
    items: {
      type: Array,
      default: () => []
    },
    navMode: {
      type: String,
      default: 'MIXED'
    }
  },
  data() {
    return {
      currentMode: 'custom',
      currentItems: [],
      currentParent: {
        id: 0,
        title: '/ (root)'
      },
      parents: [],
      loadedCache: []
    }
  },
  computed: {
    path: get('page/path'),
    locale: get('page/locale')
  },
  methods: {
    switchMode (mode) {
      this.currentMode = mode
      window.localStorage.setItem('navPref', mode)
      if (mode === `browse` && this.loadedCache.length < 1) {
        this.loadFromCurrentPath()
      }
    },
    async fetchBrowseItems (item) {
      this.$store.commit(`loadingStart`, 'browse-load')
      if (!item) {
        item = this.currentParent
      }

      if (this.loadedCache.indexOf(item.id) < 0) {
        this.currentItems = []
      }

      if (item.id === 0) {
        this.parents = []
      } else {
        const flushRightIndex = _.findIndex(this.parents, ['id', item.id])
        if (flushRightIndex >= 0) {
          this.parents = _.take(this.parents, flushRightIndex)
        }
        if (this.parents.length < 1) {
          this.parents.push(this.currentParent)
        }
        this.parents.push(item)
      }

      this.currentParent = item

      const resp = await this.$apollo.query({
        query: gql`
          query ($parent: Int, $locale: String!) {
            pages {
              tree(parent: $parent, mode: ALL, locale: $locale) {
                id
                path
                title
                isFolder
                pageId
                parent
                locale
              }
            }
          }
        `,
        fetchPolicy: 'cache-first',
        variables: {
          parent: item.id,
          locale: this.locale
        }
      })
      this.loadedCache = _.union(this.loadedCache, [item.id])
      this.currentItems = _.get(resp, 'data.pages.tree', [])
      this.$store.commit(`loadingStop`, 'browse-load')
    },
    async loadFromCurrentPath() {
      this.$store.commit(`loadingStart`, 'browse-load')
      const resp = await this.$apollo.query({
        query: gql`
          query ($path: String, $locale: String!) {
            pages {
              tree(path: $path, mode: ALL, locale: $locale, includeAncestors: true) {
                id
                path
                title
                isFolder
                pageId
                parent
                locale
              }
            }
          }
        `,
        fetchPolicy: 'cache-first',
        variables: {
          path: this.path,
          locale: this.locale
        }
      })
      const items = _.get(resp, 'data.pages.tree', [])
      const curPage = _.find(items, ['pageId', this.$store.get('page/id')])
      if (!curPage) {
        console.warn('Could not find current page in page tree listing!')
        return
      }

      let curParentId = curPage.parent
      let invertedAncestors = []
      while (curParentId) {
        const curParent = _.find(items, ['id', curParentId])
        if (!curParent) {
          break
        }
        invertedAncestors.push(curParent)
        curParentId = curParent.parent
      }

      this.parents = [this.currentParent, ...invertedAncestors.reverse()]
      this.currentParent = _.last(this.parents)

      this.loadedCache = [curPage.parent]
      this.currentItems = _.filter(items, ['parent', curPage.parent])
      this.$store.commit(`loadingStop`, 'browse-load')
    },
    goHome () {
      window.location.assign(siteLangs.length > 0 ? `/${this.locale}/home` : '/')
    }
  },
  mounted () {
    this.currentParent.title = `/ ${this.$t('common:sidebar.root')}`
    if (this.navMode === 'TREE') {
      this.currentMode = 'browse'
    } else if (this.navMode === 'STATIC') {
      this.currentMode = 'custom'
    } else {
      this.currentMode = window.localStorage.getItem('navPref') || 'custom'
    }
    if (this.currentMode === 'browse') {
      this.loadFromCurrentPath()
    }
  }
}
</script>
