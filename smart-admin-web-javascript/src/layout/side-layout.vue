<template>
  <a-layout class="admin-layout" style="min-height: 100%">
    <!-- 侧边菜单 side-menu -->
    <a-layout-sider
      :id="LAYOUT_ELEMENT_IDS.menu"
      class="side-menu"
      :width="sideMenuWidth"
      v-model:collapsed="collapsed"
      :theme="theme"
      v-show="!fullScreenFlag"
    >
      <!-- 左侧菜单 -->
      <SideMenu :collapsed="collapsed" />
    </a-layout-sider>

    <!--中间内容，一共三部分：1、顶部;2、中间内容区域;3、底部（一般是公司版权信息）;-->
    <a-layout :id="LAYOUT_ELEMENT_IDS.main" :style="`height: ${windowHeight}px`" class="admin-layout-main">
      <!-- 顶部头部信息 -->
      <a-layout-header class="layout-header" :id="LAYOUT_ELEMENT_IDS.header" v-show="!fullScreenFlag">
        <a-row class="layout-header-user" justify="space-between">
          <a-col
            class="layout-header-left"
            :style="{
              'max-width': `calc(100% - ${rightWidth}px)`,
            }"
          >
            <div class="layout-header-box">
              <span class="collapsed-button">
                <menu-unfold-outlined v-if="collapsed" class="trigger" @click="() => (collapsed = !collapsed)" />
                <menu-fold-outlined v-else class="trigger" @click="() => (collapsed = !collapsed)" />
              </span>
              <a-tooltip placement="bottom">
                <template #title>首页</template>
                <span class="home-button" @click="goHome">
                  <home-outlined class="trigger" />
                </span>
              </a-tooltip>
              <span class="location-breadcrumb">
                <PageTag v-if="pageTagLocation === 'top'" />
                <MenuLocationBreadcrumb />
              </span>
            </div>
          </a-col>
          <!---用戶操作区域-->
          <a-col class="layout-header-right">
            <HeaderUserSpace />
          </a-col>
        </a-row>
        <PageTag v-if="pageTagLocation === 'center'" />
      </a-layout-header>

      <!--中间内容-->
      <a-layout-content :id="LAYOUT_ELEMENT_IDS.content" class="admin-layout-content">
        <!--不keepAlive的iframe使用单个iframe组件-->
        <IframeIndex v-if="iframeNotKeepAlivePageFlag" :key="route.name" :name="route.name" :url="route.meta.frameUrl" />
        <!--keepAlive的iframe 每个页面一个iframe组件-->
        <IframeIndex
          v-for="item in keepAliveIframePages"
          v-show="route.name === item.name"
          :key="item.name"
          :name="item.name"
          :url="item.meta.frameUrl"
        />
        <!--非iframe使用router-view-->
        <div v-show="!iframeNotKeepAlivePageFlag && keepAliveIframePages.every((e) => route.name != e.name)">
          <router-view v-slot="{ Component }">
            <keep-alive :include="keepAliveIncludes">
              <component :is="Component" :key="route.name" />
            </keep-alive>
          </router-view>
        </div>
      </a-layout-content>

      <!-- footer 版权公司信息 -->
      <a-layout-footer class="layout-footer" v-show="footerFlag">
        <smart-footer />
      </a-layout-footer>
      <!--- 回到顶部 -->
      <a-back-top :target="backTopTarget" :visibilityHeight="80" />
    </a-layout>
    <!-- 右侧帮助文档 help-doc -->
    <a-layout-sider
      v-if="helpDocFlag"
      v-show="helpDocExpandFlag"
      theme="light"
      :width="180"
      class="help-doc-sider"
      :trigger="null"
      style="min-height: 100%"
    >
      <SideHelpDoc />
    </a-layout-sider>
  </a-layout>
</template>

<script setup>
  import { computed, onMounted, ref, watch } from 'vue';
  import { useAppConfigStore } from '../store/modules/system/app-config';
  import HeaderUserSpace from './components/header-user-space/index.vue';
  import MenuLocationBreadcrumb from './components/menu-location-breadcrumb/index.vue';
  import PageTag from './components/page-tag/index.vue';
  import SideMenu from './components/side-menu/index.vue';
  import SmartFooter from './components/smart-footer/index.vue';
  import { smartKeepAlive } from './components/smart-keep-alive';
  import IframeIndex from '/@/components/framework/iframe/iframe-index.vue';
  import watermark from '../lib/smart-watermark';
  import { useUserStore } from '/@/store/modules/system/user';
  import SideHelpDoc from './components/side-help-doc/index.vue';
  import { useRouter } from 'vue-router';
  import { HOME_PAGE_NAME } from '/@/constants/system/home-const';
  import { LAYOUT_ELEMENT_IDS } from '/@/layout/layout-const.js';
  import { nextTick } from 'vue';
  const appConfigStore = useAppConfigStore();

  const windowHeight = ref(window.innerHeight);
  //是否全屏
  const fullScreenFlag = computed(() => useAppConfigStore().$state.fullScreenFlag);
  //菜单宽度
  const sideMenuWidth = computed(() => useAppConfigStore().$state.sideMenuWidth);
  //主题颜色
  const theme = computed(() => useAppConfigStore().$state.sideMenuTheme);
  //是否显示标签页
  const pageTagFlag = computed(() => useAppConfigStore().$state.pageTagFlag);
  // 是否显示帮助文档
  const helpDocFlag = computed(() => useAppConfigStore().$state.helpDocFlag);
  // 是否默认展开帮助文档
  const helpDocExpandFlag = computed(() => useAppConfigStore().$state.helpDocExpandFlag);
  // 是否显示页脚
  const footerFlag = computed(() => useAppConfigStore().$state.footerFlag);
  // 是否显示水印
  const watermarkFlag = computed(() => useAppConfigStore().$state.watermarkFlag);
  // 标签页位置
  const pageTagLocation = computed(() => useAppConfigStore().$state.pageTagLocation);
  // 多余高度
  const dueHeight = computed(() => {
    let due = 40;
    if (useAppConfigStore().$state.pageTagFlag) {
      due = due + 40;
    }
    if (useAppConfigStore().$state.footerFlag) {
      due = due + 40;
    }
    return due;
  });

  watch(
    pageTagLocation,
    (newVal) => {
      if (newVal == 'top') {
        nextTick(() => {
          sizeComputed();
        });
      }
    },
    {
      immediate: true,
    }
  );

  const rightWidth = ref(0);
  function sizeComputed() {
    const tagParentElement = document.querySelector('.location-breadcrumb');
    const tagsElement = tagParentElement.querySelector('.ant-tabs-nav-list');
    const parentElement = document.querySelector('.layout-header-user');
    const rightElement = document.querySelector('.layout-header-right');
    rightWidth.value = rightElement.offsetWidth;
    let ro = new ResizeObserver((e) => {
      rightWidth.value = rightElement.offsetWidth + 10;
    });
    ro.observe(rightElement);
    ro.observe(tagsElement);
    ro.observe(parentElement);
  }

  //是否隐藏菜单
  const collapsed = ref(false);

  //页面初始化的时候加载水印
  onMounted(() => {
    if (watermarkFlag.value) {
      watermark.set(LAYOUT_ELEMENT_IDS.content, useUserStore().actualName);
    } else {
      watermark.clear();
    }
  });

  watch(
    () => watermarkFlag.value,
    (newValue) => {
      if (newValue) {
        watermark.set(LAYOUT_ELEMENT_IDS.content, useUserStore().actualName);
      } else {
        watermark.clear();
      }
    }
  );

  //回到顶部
  const backTopTarget = () => {
    return document.getElementById(LAYOUT_ELEMENT_IDS.main);
  };

  const router = useRouter();
  function goHome() {
    router.push({ name: HOME_PAGE_NAME });
  }

  window.addEventListener('resize', function () {
    windowHeight.value = window.innerHeight;
  });

  // ----------------------- keep-alive相关 -----------------------
  let { route, keepAliveIncludes, iframeNotKeepAlivePageFlag, keepAliveIframePages } = smartKeepAlive();
</script>

<style lang="less" scoped>
  :deep(.ant-layout-header) {
    height: auto;
  }

  :deep(.layout-header) {
    height: auto;
  }

  .layout-header {
    background: #fff;
    padding: 0;
    z-index: 21;
  }

  .layout-header-user {
    height: @header-user-height;
    border-bottom: 1px solid #f6f6f6;
  }

  .layout-header-left {
    display: flex;
    height: @header-user-height;

    .layout-header-box {
      width: 100%;
      overflow: hidden;
      display: flex;
    }
    .collapsed-button {
      margin-left: 10px;
      line-height: @header-user-height;
    }

    .home-button {
      margin-left: 15px;
      cursor: pointer;
      padding: 0 5px;
      line-height: @header-user-height;
    }

    .home-button:hover {
      background-color: #efefef;
    }

    .location-breadcrumb {
      width: calc(100% - 56px);
      margin-left: 15px;
      line-height: @header-user-height;
    }
  }

  .layout-header-right {
    display: flex;
    height: @header-user-height;
  }

  .layout-container {
    height: calc(100vh - @header-height);
    overflow-x: hidden;
  }

  .admin-layout {
    .side-menu {
      height: 100vh;
      overflow-x: hidden;
      overflow-y: scroll;
      &.fixed-side {
        position: fixed;
        height: 100vh;
        left: 0;
        top: 0;
      }
    }

    .side-menu::-webkit-scrollbar {
      width: 4px;
    }
    .side-menu::-webkit-scrollbar-thumb {
      border-radius: 10px;
      background: rgba(0, 0, 0, 0.2);
    }
    .side-menu::-webkit-scrollbar-track {
      border-radius: 0;
      background: rgba(0, 0, 0, 0.1);
    }

    .help-doc-sider {
      flex: 0 !important;
      min-width: 100px;
      height: 100vh;
      max-width: 100px;
      width: auto !important;

      &.fixed-side {
        position: fixed;
        height: 100vh;
        right: 0;
        top: 0;
      }
    }

    .virtual-side {
      transition: all 0.2s;
    }

    .virtual-header {
      transition: all 0.2s;
      opacity: 0;

      &.fixed-tabs.multi-page:not(.fixed-header) {
        height: 0;
      }
    }

    .admin-layout-main {
      overflow-y: hidden;
      overflow-x: hidden;
    }

    .admin-layout-content {
      background-color: inherit;
      min-height: auto;
      position: relative;
      overflow-x: hidden;
      padding: 10px 10px 0px 10px;
      height: calc(100% - v-bind(dueHeight) px);
    }
  }

  .layout-footer {
    position: relative;
    padding: 7px 0px;
    display: flex;
    justify-content: center;
  }
</style>
