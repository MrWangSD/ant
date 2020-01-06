<template>
<div>
  <a-layout :class="['layout', device]">
    <!-- SideMenu -->
    <a-drawer
      v-if="isMobile()"
      placement="left"
      :wrapClassName="`drawer-sider ${navTheme}`"
      :closable="false"
      :visible="collapsed"
      @close="drawerClose"
    >
      <side-menu
        mode="inline"
        :menus="menus"
        :theme="navTheme"
        :collapsed="false"
        :collapsible="true"
        @menuSelect="selectDrawerMenu"
      ></side-menu>
    </a-drawer>

    <side-menu
      v-else-if="!isMobile() && sideMenus.length > 0"
      mode="inline"
      :menus="sideMenus"
      :theme="navTheme"
      :collapsed="collapsed"
      :collapsible="true"
    ></side-menu>

    <a-layout :class="['sidemenu', `content-width-${contentWidth}`]" :style="{ paddingLeft: contentPaddingLeft, minHeight: '100vh' }">
      <!-- layout header -->
      <global-header
        mode="sidemenu"
        :menus="topMenus"
        :theme="navTheme"
        :collapsed="collapsed"
        :hideCollapsed="hideCollapsed"
        :device="device"
        @toggle="toggle"
        @topMenuSelect="selectTopMenu"
        @initSideMenu="initSideMenu"
      />

      <!-- layout content -->
      <a-layout-content :style="{ height: '100%', margin: '24px 24px 0', paddingTop: fixedHeader ? '64px' : '0' }">
        <multi-tab v-if="multiTab"></multi-tab>
        <transition name="page-transition">
          <route-view />
        </transition>
      </a-layout-content>

      <!-- layout footer -->
      <a-layout-footer>
        <global-footer />
      </a-layout-footer>

      <!-- Setting Drawer (show in development mode) -->
      <setting-drawer v-if="!production"></setting-drawer>
    </a-layout>
  </a-layout>
  <div class="ant-back-top-inner">
    <a-back-top>
      <div class="ant-back-top-inner" :style="{background: primaryColor}">
        <a-icon type="up" :style="{fontSize: '28px', color: '#fff'}" />
      </div>
    </a-back-top>
  </div>
</div>
</template>

<script>
import { triggerWindowResizeEvent } from '@/utils/util'
import { mapState, mapActions } from 'vuex'
import { mixin, mixinDevice } from '@/utils/mixin'
import config from '@/config/defaultSettings'

import RouteView from './RouteView'
import MultiTab from '@/components/MultiTab'
import SideMenu from '@/components/Menu/SideMenu'
import GlobalHeader from '@/components/GlobalHeader'
import GlobalFooter from '@/components/GlobalFooter'
import SettingDrawer from '@/components/SettingDrawer'

export default {
  name: 'BasicLayout',
  mixins: [mixin, mixinDevice],
  components: {
    RouteView,
    MultiTab,
    SideMenu,
    GlobalHeader,
    GlobalFooter,
    SettingDrawer
  },
  data () {
    return {
      production: config.production,
      collapsed: false,
      menus: [],
      topMenus: [],
      sideMenus: []
    }
  },
  computed: {
    ...mapState({
      // 动态主路由
      mainMenu: state => state.permission.addRouters
    }),
    contentPaddingLeft () {
      if (!this.fixSidebar || this.isMobile()) {
        return '0'
      }
      if (this.sidebarOpened) {
        return '256px'
      }
      return '80px'
    },
    hideCollapsed () {
      return this.sideMenus.length === 0
    }
  },
  watch: {
    sidebarOpened (val) {
      this.collapsed = !val
    }
  },
  created () {
    this.menus = this.mainMenu.find(item => item.path === '/').children
    this.topMenus = this.menus.map(item => ({
      component: item.component,
      meta: item.meta,
      name: item.name,
      path: item.path
    }))
    // console.log(this.menus)
    this.collapsed = !this.sidebarOpened
  },
  mounted () {
    const userAgent = navigator.userAgent
    if (userAgent.indexOf('Edge') > -1) {
      this.$nextTick(() => {
        this.collapsed = !this.collapsed
        setTimeout(() => {
          this.collapsed = !this.collapsed
        }, 16)
      })
    }
  },
  methods: {
    ...mapActions(['setSidebar']),
    toggle () {
      this.collapsed = !this.collapsed
      this.setSidebar(!this.collapsed)
      triggerWindowResizeEvent()
    },
    paddingCalc () {
      let left = ''
      if (this.sidebarOpened) {
        left = this.isDesktop() ? '256px' : '80px'
      } else {
        left = (this.isMobile() && '0') || ((this.fixSidebar && '80px') || '0')
      }
      return left
    },
    selectDrawerMenu () {
      if (!this.isDesktop()) {
        this.collapsed = false
      }
    },
    // 选择顶部一级菜单
    selectTopMenu(menu) {
      // console.log(menu)
      let thisTopMenu = this.menus.filter(item => item.path === menu.key)
      if(thisTopMenu){
        this.sideMenus = thisTopMenu.length > 0 && thisTopMenu[0].children ? thisTopMenu[0].children : []
        this.collapsed = false;
      }
    },
    // 初始化一级菜单 刷新情况
    initSideMenu(seleckedKeys){
      let that = this
      if(seleckedKeys.length > 0){
        let selectedKey = seleckedKeys[0];
        let isFind = false;
        // 迭代韩式
        let findKey = (menu, topKey) => {
          // console.log(menu.path)
          if(!isFind){
            if(menu.path === selectedKey){
              // 找到一级菜单-设置二级菜单
              let thisTopMenu = that.menus.filter(item => item.path === topKey)
              if(thisTopMenu){
                that.sideMenus = thisTopMenu.length > 0 && thisTopMenu[0].children ? thisTopMenu[0].children : []
                that.collapsed = false;
              }
              isFind = true;
            }else if(menu.children && menu.children.length > 0){
              menu.children.map(child => {
                findKey(child, topKey);
              })
            }
          }
        }
        // 查找
        that.menus.map(menu => {
          findKey(menu, menu.path);
        })
      }
    },
    drawerClose () {
      this.collapsed = false
    }
  }
}
</script>

<style lang="less">
@import url('../components/global.less');

/*
 * The following styles are auto-applied to elements with
 * transition="page-transition" when their visibility is toggled
 * by Vue.js.
 *
 * You can easily play with the page transition by editing
 * these styles.
 */
 .ant-back-top-inner{
  & /deep/ .ant-back-top {
    right: 0px;
    bottom: 100px;
  }
  .ant-back-top-inner {
    height: 40px;
    width: 40px;
    line-height: 40px;
    border-top-left-radius: 4px;
    border-bottom-left-radius: 4px;
    color: #fff;
    text-align: center;
    font-size: 20px;
  }
 } 

.page-transition-enter {
  opacity: 0;
}

.page-transition-leave-active {
  opacity: 0;
}

.page-transition-enter .page-transition-container,
.page-transition-leave-active .page-transition-container {
  -webkit-transform: scale(1.1);
  transform: scale(1.1);
}
</style>
