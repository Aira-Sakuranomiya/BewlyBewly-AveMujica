<script setup lang="ts">
import { onKeyStroke, useMouseInElement } from '@vueuse/core'
import type { Ref, UnwrapNestedRefs } from 'vue'

import { useBewlyApp } from '~/composables/useAppProvider'
import { useDelayedHover } from '~/composables/useDelayedHover'
import { OVERLAY_SCROLL_BAR_SCROLL, TOP_BAR_VISIBILITY_CHANGE } from '~/constants/globalEvents'
import { AppPage } from '~/enums/appEnums'
import { settings } from '~/logic'
import api from '~/utils/api'
import { getUserID, isHomePage, removeHttpFromUrl } from '~/utils/main'
import emitter from '~/utils/mitt'

import ChannelsPop from './components/ChannelsPop.vue'
import FavoritesPop from './components/FavoritesPop.vue'
import HistoryPop from './components/HistoryPop.vue'
import MomentsPop from './components/MomentsPop.vue'
import MorePop from './components/MorePop.vue'
import NotificationsPop from './components/NotificationsPop.vue'
import UploadPop from './components/UploadPop.vue'
import WatchLaterPop from './components/WatchLaterPop.vue'
import { updateInterval } from './notify'
import OldUserPanelPop from './oldTopBarComponents/OldUserPanelPop.vue'
import type { UnReadDm, UnReadMessage, UserInfo } from './types'

// import { useTopBarStore } from '~/stores/topBarStore'

// const popups = { NotificationsPop, MomentsPop, FavoritesPop, HistoryPop }

// const topBarStore = useTopBarStore()

// const topBarItems = computed(() => {
//   return topBarStore.topBarItems
// })

const { activatedPage, scrollbarRef, reachTop } = useBewlyApp()

const mid = getUserID() || ''
const userInfo = reactive<UserInfo | NonNullable<unknown>>({}) as UnwrapNestedRefs<UserInfo>

const hideTopBar = ref<boolean>(false)
const headerTarget = ref(null)
const { isOutside: isOutsideTopBar } = useMouseInElement(headerTarget)

// initially, assume the user is logged in cuz data retrieval is slow, which may show the login
// button even after login. if the user is not logged in, the login button will show up later
const isLogin = ref<boolean>(true)

const logo = ref<HTMLElement>() as Ref<HTMLElement>
const avatarImg = ref<HTMLImageElement>() as Ref<HTMLImageElement>
const avatarShadow = ref<HTMLImageElement>() as Ref<HTMLImageElement>
const favoritesPopRef = ref<any>()
const momentsPopRef = ref()

const scrollTop = ref<number>(0)
const oldScrollTop = ref<number>(0)

const isSearchPage = computed(() => {
  if (/https?:\/\/search.bilibili.com\/.*$/.test(location.href))
    return true
  return false
})

const showSearchBar = computed(() => {
  if (isHomePage()) {
    if (settings.value.useOriginalBilibiliHomepage)
      return true
    if (activatedPage.value === AppPage.Search)
      return false
    if (settings.value.useSearchPageModeOnHomePage && activatedPage.value === AppPage.Home && reachTop.value)
      return false
  }
  else {
    if (isSearchPage.value)
      return false
  }
  return true
})

const isTopBarFixed = computed(() => {
  if (
    (isHomePage() && settings.value.useOriginalBilibiliHomepage)
    // video page
    || /https?:\/\/(?:www.)?bilibili.com\/(?:video|list)\/.*/.test(location.href)
    // anime playback & movie page
    || /https?:\/\/(?:www.)?bilibili.com\/bangumi\/play\/.*/.test(location.href)
    // moment page
    || /https?:\/\/t.bilibili.com.*/.test(location.href)
    // channel, anime, chinese anime, tv shows, movie, variety shows, mooc
    || /https?:\/\/(?:www.)?bilibili.com\/(?:v|anime|guochuang|tv|movie|variety|mooc).*/.test(location.href)
    // articles page
    || /https?:\/\/(?:www.)?bilibili.com\/read\/home.*/.test(location.href)
  ) {
    return true
  }
  return false
})

const showTopBar = computed((): boolean => {
  if (
    // Creative center page
    /https?:\/\/member.bilibili.com\/platform.*/.test(location.href)
    // https://github.com/BewlyBewly/BewlyBewly/issues/1276
    || /https?:\/\/(?:www\.)?bilibili\.com\/read\/(?:preview|pcpreview).*/.test(location.href)
  ) {
    return false
  }
  if (settings.value.showTopBar)
    return true
  return false
})

// #region Popups visibility control
const popupVisible = reactive({
  channels: false,
  userPanel: false,
  notifications: false,
  moments: false,
  favorites: false,
  history: false,
  watchLater: false,
  upload: false,
  more: false,
})

function closeAllTopBarPopup(exceptionKey?: keyof typeof popupVisible) {
  Object.keys(popupVisible).forEach((key) => {
    if (key !== exceptionKey)
      popupVisible[key as keyof typeof popupVisible] = false
  })
}

const rightSideInactive = computed(() => {
  let isInactive = false
  Object.entries(popupVisible).forEach(([key, value]) => {
    if (value && key !== 'userPanel') {
      isInactive = true
    }
  })
  return isInactive
})

// Channels
const channels = useDelayedHover({
  beforeEnter: () => closeAllTopBarPopup('channels'),
  enter: () => {
    popupVisible.channels = true
  },
  leave: () => {
    popupVisible.channels = false
  },
})

// Avatar
const avatar = useDelayedHover({
  beforeEnter: () => closeAllTopBarPopup('userPanel'),
  enter: () => {
    popupVisible.userPanel = true
  },
  beforeLeave: () => {
    popupVisible.userPanel = false
  },
  leave: () => {
    popupVisible.userPanel = false
  },
})

// Notifications
const notifications = useDelayedHover({
  beforeEnter: () => closeAllTopBarPopup('notifications'),
  enter: () => {
    popupVisible.notifications = true
  },
  leave: () => {
    popupVisible.notifications = false
  },
})

// Moments
const moments = useDelayedHover({
  beforeEnter: () => closeAllTopBarPopup('moments'),
  enter: () => {
    popupVisible.moments = true
    momentsPopRef.value && momentsPopRef.value.checkIfHasNewMomentsThenUpdateMoments() // eslint-disable-line ts/no-unused-expressions
  },
  leave: () => {
    popupVisible.moments = false
  },
})

// Favorites
const favorites = useDelayedHover({
  beforeEnter: () => closeAllTopBarPopup('favorites'),
  enter: () => {
    popupVisible.favorites = true
  },
  leave: () => {
    popupVisible.favorites = false
  },
})

// History
const history = useDelayedHover({
  beforeEnter: () => closeAllTopBarPopup('history'),
  enter: () => {
    popupVisible.history = true
  },
  leave: () => {
    popupVisible.history = false
  },
})

// Watch Later
const watchLater = useDelayedHover({
  beforeEnter: () => closeAllTopBarPopup('watchLater'),
  enter: () => {
    popupVisible.watchLater = true
  },
  leave: () => {
    popupVisible.watchLater = false
  },
})

// Upload
const upload = useDelayedHover({
  beforeEnter: () => closeAllTopBarPopup('upload'),
  enter: () => {
    popupVisible.upload = true
  },
  leave: () => {
    popupVisible.upload = false
  },
})

// More
const more = useDelayedHover({
  beforeEnter: () => closeAllTopBarPopup(),
  enter: () => {
    popupVisible.more = true
  },
  leave: () => popupVisible.more = false,
})

// #endregion

watch(() => settings.value.autoHideTopBar, (newVal) => {
  if (!newVal)
    toggleTopBarVisible(true)
})

watch(
  () => popupVisible.notifications,
  (newVal, oldVal) => {
    // Stop loading new message counts on the message page, because it resets to 0 when the
    // users reads the messages on this page
    if (oldVal === undefined && /https?:\/\/message.bilibili.com.*$/.test(location.href))
      return

    if (newVal === oldVal)
      return

    if (!newVal)
      getUnreadMessageCount()
  },
  { immediate: true },
)

watch(
  () => popupVisible.moments,
  async (newVal, oldVal) => {
    if (newVal === oldVal)
      return

    if (!newVal)
      await getTopBarNewMomentsCount()
  },
  { immediate: true },
)

watch(() => popupVisible.favorites, (newVal, oldVal) => {
  if (newVal === oldVal)
    return
  if (newVal) {
    nextTick(() => {
      if (favoritesPopRef.value)
        favoritesPopRef.value.refreshFavoriteResources()
    })
  }
})

watch(activatedPage, () => {
  toggleTopBarVisible(true)
})

onMounted(() => {
  initData()
  nextTick(() => {
    toggleTopBarVisible(true)

    emitter.off(OVERLAY_SCROLL_BAR_SCROLL)
    if (isHomePage() && !settings.value.useOriginalBilibiliHomepage) {
      emitter.on(OVERLAY_SCROLL_BAR_SCROLL, () => {
        handleScroll()
      })
    }
    else {
      window.addEventListener('scroll', handleScroll)
    }
  })
})

onUnmounted(() => {
  window.removeEventListener('scroll', handleScroll)
  emitter.off(OVERLAY_SCROLL_BAR_SCROLL)
})

async function initData() {
  await getUserInfo()

  // automatically update notifications and moments count
  setInterval(() => {
    getUnreadMessageCount()
    getTopBarNewMomentsCount()
  }, updateInterval)
}

function handleScroll() {
  if (isHomePage() && !settings.value.useOriginalBilibiliHomepage) {
    const osInstance = scrollbarRef.value?.osInstance()
    scrollTop.value = osInstance.elements().viewport.scrollTop as number
  }
  else {
    scrollTop.value = document.documentElement.scrollTop
  }

  if (scrollTop.value === 0)
    toggleTopBarVisible(true)

  if (settings.value.autoHideTopBar && isOutsideTopBar && scrollTop.value !== 0) {
    if (scrollTop.value > oldScrollTop.value)
      toggleTopBarVisible(false)

    else
      toggleTopBarVisible(true)
  }

  oldScrollTop.value = scrollTop.value
}

async function getUserInfo() {
  try {
    const res = await api.user.getUserInfo()

    if (res.code === 0) {
      isLogin.value = true
      Object.assign(userInfo, res.data)
    }
    // Account not logged in
    else if (res.code === -101) {
      isLogin.value = false
    }
  }
  catch (error) {
    isLogin.value = false
    console.error(error)
  }
}

// #region Notifications
const unReadMessage = reactive<UnReadMessage | NonNullable<unknown>>(
  {},
) as UnwrapNestedRefs<UnReadMessage>
const unReadDm = reactive<UnReadDm>({} as UnwrapNestedRefs<UnReadDm>)
const unReadMessageCount = ref<number>(0)

async function getUnreadMessageCount() {
  if (!isLogin.value)
    return

  let result = 0

  try {
    let res
    res = await api.notification.getUnreadMsg()
    if (res.code === 0) {
      Object.assign(unReadMessage, res.data)
      Object.entries(unReadMessage).forEach(([key, value]) => {
        if (key !== 'up' && key !== 'recv_reply' && key !== 'recv_like') {
          if (typeof value === 'number')
            result += value
        }
      })
    }

    res = await api.notification.getUnreadDm()
    if (res.code === 0) {
      Object.assign(unReadDm, res.data)
      if (typeof unReadDm.follow_unread === 'number')
        result += unReadDm.follow_unread
    }
  }
  catch (error) {
    console.error(error)
  }
  finally {
    unReadMessageCount.value = result
  }
}
// #endregion

// #region Moments
const newMomentsCount = ref<number>(0)

async function getTopBarNewMomentsCount() {
  if (!isLogin.value)
    return

  let result = 0

  try {
    const res = await api.moment.getTopBarNewMomentsCount()
    if (res.code === 0) {
      if (typeof res.data.update_info.item.count === 'number')
        result = res.data.update_info.item.count
    }
  }
  finally {
    newMomentsCount.value = result
  }
}
// #endregion

// https://github.com/BewlyBewly/BewlyBewly/issues/1220
onKeyStroke('/', () => {
  toggleTopBarVisible(true)
})

function toggleTopBarVisible(visible: boolean) {
  hideTopBar.value = !visible
  emitter.emit(TOP_BAR_VISIBILITY_CHANGE, visible)
}

defineExpose({
  toggleTopBarVisible,
  handleScroll,
})
</script>

<template>
  <Transition name="top-bar">
    <header
      v-if="showTopBar"
      ref="headerTarget"
      w="full" transition="all 300 ease-in-out"
      :class="{ hide: hideTopBar }"
      :style="{ position: isTopBarFixed ? 'fixed' : 'absolute' }"
    >
      <main
        max-w="$bew-page-max-width"
        flex="~ justify-between items-center gap-4"
        p="x-12" m-auto
        h="$bew-top-bar-height"
      >
        <!-- Top bar mask -->
        <div
          v-if="!reachTop"
          style="
            mask-image: linear-gradient(to bottom,  black 20%, transparent);
          "
          :style="{ backdropFilter: settings.disableFrostedGlass ? 'none' : 'blur(4px)' }"
          pos="absolute top-0 left-0" w-full h-80px
          pointer-events-none transform-gpu
        />
        <Transition name="fade">
          <div
            v-if="!reachTop"
            pos="absolute top-0 left-0" w-full h-80px
            pointer-events-none opacity-80
            :style="{
              background: `linear-gradient(to bottom, ${(
                settings.wallpaper
                || settings.useSearchPageModeOnHomePage
                && settings.searchPageWallpaper
                && settings.individuallySetSearchPageWallpaper)
                && isHomePage()
                ? 'rgba(0,0,0,.6)' : `${isHomePage() ? 'var(--bew-homepage-bg)' : 'var(--bew-bg)'}`}, transparent)`,
            }"
          />
        </Transition>

        <div shrink-0 flex="inline xl:1 justify-center">
          <div
            ref="channels"
            z-1 relative w-fit mr-auto
          >
            <a
              ref="logo" href="//www.bilibili.com"
              target="_top"
              class="group logo"
              :class="{ activated: popupVisible.channels }"
              style="backdrop-filter: var(--bew-filter-glass-1);"
              grid="~ place-items-center" border="1 $bew-border-color"
              rounded="46px" duration-300
              bg="$bew-elevated hover:$bew-theme-color dark-hover:white"
              shadow="$bew-shadow-2"
              w-46px h-46px transform-gpu
            >

              <svg
                t="1720198072316" class="icon" viewBox="0 0 1024 1024" version="1.1" xmlns="http://www.w3.org/2000/svg"
                p-id="1477" width="38" height="38"
                un-fill="$bew-theme-color dark:white" un-group-hover:fill="white dark:$bew-theme-color"
                duration-300 mt--1px
              >
                >
                <path d="M450.803484 456.506027l-120.670435 23.103715 10.333298 45.288107 119.454151-23.102578-9.117014-45.289244z m65.04448 120.060586c-29.483236 63.220622-55.926329 15.502222-55.926328 15.502223l-19.754098 12.768142s38.90176 53.192249 75.986489 12.764729c43.770311 40.42752 77.203911-13.068516 77.203911-13.068516l-17.934791-11.55072c0.001138-0.304924-31.305956 44.983182-59.575183-16.415858z m59.57632-74.773617L695.182222 524.895573l10.029511-45.288106-120.364373-23.103716-9.423076 45.289245z m237.784178-88.926436c-1.905778-84.362809-75.487004-100.540871-75.487004-100.540871s-57.408853-0.316302-131.944676-0.95232l54.237867-52.332089s8.562916-10.784996-6.026809-22.834062c-14.592-12.051342-15.543182-6.660551-20.615396-3.487289-4.441884 3.169849-69.462471 66.920676-80.878933 78.340551-29.494613 0-60.2624-0.319716-90.075591-0.319716h10.466418s-77.705671-76.754489-82.781298-80.241777c-5.075627-3.488427-5.709369-8.56064-20.616533 3.487289-14.589724 12.05248-6.026809 22.8352-6.026809 22.8352l55.504213 53.919288c-60.261262 0-112.280462 0.319716-136.383147 1.268623-78.025387 22.521173-71.99744 100.859449-71.99744 100.859449s0.950044 168.100978 0 253.103217c8.562916 85.00224 73.899804 98.636231 73.899805 98.636231s26.007324 0.63488 45.357511 0.63488c1.900089 5.391929 3.486151 32.034133 33.302756 32.034134 29.495751 0 33.30048-32.034133 33.30048-32.034134s217.263218-0.950044 235.340231-0.950044c0.953458 9.196658 5.394204 33.619058 35.207395 33.303893 29.494613-0.636018 31.714418-35.20512 31.714418-35.20512s10.151253-0.95232 40.280747 0c70.413653-13.005938 74.534684-95.468658 74.534684-95.468657s-1.265209-169.689316-0.312889-254.056676zM752.628622 681.8304c0 13.319964-10.467556 24.102684-23.471218 24.102684H300.980907c-13.003662 0-23.47008-10.78272-23.47008-24.102684V397.961671c0-13.32224 10.467556-24.106098 23.47008-24.106098h428.176497c13.003662 0 23.471218 10.783858 23.471218 24.106098v283.868729z" p-id="1478" /></svg>
            </a>

            <Transition name="slide-in">
              <ChannelsPop
                v-show="popupVisible.channels"
                class="bew-popover"
                pos="!left-0 !top-70px"
                transform="!translate-x-0"
              />
            </Transition>
          </div>
        </div>

        <!-- search bar -->
        <div flex="inline 1 md:justify-center <md:justify-end items-center" w="full">
          <Transition name="slide-out">
            <SearchBar
              v-if="showSearchBar"
              style="
              --b-search-bar-normal-color: var(--bew-elevated);
              --b-search-bar-hover-color: var(--bew-elevated-hover);
            "
            />
          </Transition>
        </div>

        <!-- right content -->
        <div
          class="right-side"
          flex="inline xl:1 justify-end items-center"
        >
          <!-- Avatar -->
          <div
            v-if="isLogin"
            ref="avatar"
            class="avatar right-side-item relative"
            shadow="$bew-shadow-2" rounded-full
          >
            <ALink
              ref="avatarImg"
              :href="`https://space.bilibili.com/${mid}`"
              type="topBar"
              class="avatar-img"
              :class="{ hover: popupVisible.userPanel }"
              :style="{
                backgroundImage: `url(${userInfo.face ? removeHttpFromUrl(userInfo.face) : ''})`,
              }"
              rounded-full w-40px h-40px
              shadow="$bew-shadow-2"
              bg="$bew-fill-3 cover center"
              z-1
            />
            <div
              ref="avatarShadow"
              class="avatar-shadow"
              :class="{ hover: popupVisible.userPanel }"
              :style="{
                backgroundImage: `url(${userInfo.face ? removeHttpFromUrl(userInfo.face) : ''})`,
              }"
              pos="absolute top-0" z-0 pointer-events-none
              bg="cover center" blur-sm
              rounded-full
              w-40px h-40px
            />
            <svg
              v-if="userInfo.vip?.status === 1"
              class="vip-img"
              :class="{ hover: popupVisible.userPanel }"
              :style="{ opacity: popupVisible.userPanel ? 1 : 0 }"
              bg="[url(https://i0.hdslb.com/bfs/seed/jinkela/short/user-avatar/big-vip.svg)] cover no-repeat"
              w="27.5%" h="27.5%" z-1
              pos="absolute bottom-0 right-0" duration-300
            />
            <Transition name="slide-in">
              <OldUserPanelPop
                v-if="popupVisible.userPanel"
                :user-info="userInfo"
                after:h="!0"
                class="bew-popover"
              />
            </Transition>
          </div>

          <div
            class="others"
            :class="{ inactive: rightSideInactive }"
            style="
              backdrop-filter: var(--bew-filter-glass-1);
              box-shadow: var(--bew-shadow-edge-glow-1), var(--bew-shadow-2);
            "
            flex h-46px px-5px bg="$bew-elevated"
            transition="transition-property-colors duration-150"
            text="$bew-text-1" border="1 $bew-border-color" rounded-full
            transform-gpu
          >
            <div v-if="!isLogin" class="right-side-item">
              <a href="https://passport.bilibili.com/login" class="login">
                <div i-ic:outline-account-circle class="text-xl mr-2" />{{
                  $t('topbar.sign_in')
                }}
              </a>
            </div>
            <template v-if="isLogin">
              <!-- TODO: need to refactor to this -->
              <!-- <div display="lg:flex none">
              <div v-for="item in topBarItems" :key="item.i18nKey" class="right-side-item">
                <a :href="item.url" target="_blank" :title="$t(item.i18nKey)">
                  <Icon :icon="item.icon" />
                </a>

                <Component
                  :is="popups[item.popup] ?? 'div'"
                  v-if="item.popup"
                  class="bew-popover"
                />
              </div>
            </div> -->

              <!-- TODO: need to refactor to above code -->
              <div class="hidden lg:flex" gap-1>
                <!-- Notifications -->
                <div
                  ref="notifications"
                  class="right-side-item"
                  :class="{ active: popupVisible.notifications }"
                >
                  <template v-if="unReadMessageCount > 0">
                    <div
                      v-if="settings.topBarIconBadges === 'number'"
                      class="unread-num-dot"
                    >
                      {{ unReadMessageCount > 99 ? '99+' : unReadMessageCount }}
                    </div>
                    <div
                      v-else-if="settings.topBarIconBadges === 'dot'"
                      class="unread-dot"
                    />
                  </template>
                  <ALink
                    href="https://message.bilibili.com"
                    :title="$t('topbar.notifications')"
                    type="topBar"
                  >
                    <div i-tabler:bell />
                  </ALink>

                  <Transition name="slide-in">
                    <NotificationsPop
                      v-if="popupVisible.notifications"
                      class="bew-popover"
                      :un-read-message="unReadMessage"
                      :un-read-dm="unReadDm"
                    />
                  </Transition>
                </div>

                <!-- Moments -->
                <div
                  ref="moments"
                  class="right-side-item"
                  :class="{ active: popupVisible.moments }"
                >
                  <template v-if="newMomentsCount > 0">
                    <div
                      v-if="settings.topBarIconBadges === 'number'"
                      class="unread-num-dot"
                    >
                      {{ newMomentsCount > 99 ? '99+' : newMomentsCount }}
                    </div>
                    <div
                      v-else-if="settings.topBarIconBadges === 'dot'"
                      class="unread-dot"
                    />
                  </template>
                  <ALink
                    href="https://t.bilibili.com"
                    :title="$t('topbar.moments')"
                    type="topBar"
                  >
                    <div i-tabler:windmill />
                  </ALink>

                  <Transition name="slide-in">
                    <MomentsPop v-show="popupVisible.moments" ref="momentsPopRef" class="bew-popover" />
                  </Transition>
                </div>

                <!-- Favorites -->
                <div
                  ref="favorites"
                  class="right-side-item"
                  :class="{ active: popupVisible.favorites }"
                >
                  <ALink
                    :href="`https://space.bilibili.com/${mid}/favlist`"
                    :title="$t('topbar.favorites')"
                    type="topBar"
                  >
                    <div i-mingcute:star-line />
                  </ALink>

                  <Transition name="slide-in">
                    <KeepAlive>
                      <FavoritesPop
                        v-if="popupVisible.favorites"
                        ref="favoritesPopRef"
                        class="bew-popover"
                        ml--20px
                      />
                    </KeepAlive>
                  </Transition>
                </div>

                <!-- History -->
                <div
                  ref="history"
                  class="right-side-item"
                  :class="{ active: popupVisible.history }"
                >
                  <ALink
                    href="https://www.bilibili.com/account/history"
                    :title="$t('topbar.history')"
                    type="topBar"
                  >
                    <div i-mingcute:time-line />
                  </ALink>

                  <Transition name="slide-in">
                    <HistoryPop
                      v-if="popupVisible.history"
                      class="bew-popover"
                      ml--20px
                    />
                  </Transition>
                </div>

                <!-- Watch later -->
                <div
                  ref="watchLater"
                  class="right-side-item"
                  :class="{ active: popupVisible.watchLater }"
                >
                  <ALink
                    href="https://www.bilibili.com/watchlater/#/list"
                    :title="$t('topbar.watch_later')"
                    type="topBar"
                  >
                    <div i-mingcute:carplay-line />
                  </ALink>

                  <Transition name="slide-in">
                    <WatchLaterPop
                      v-if="popupVisible.watchLater"
                      class="bew-popover"
                      ml--60px
                    />
                  </Transition>
                </div>

                <!-- Creative center -->
                <div class="right-side-item">
                  <a
                    href="https://member.bilibili.com/platform/home"
                    target="_blank"
                    :title="$t('topbar.creative_center')"
                  >
                    <div i-mingcute:bulb-line />
                  </a>
                </div>
              </div>

              <!-- More -->
              <div
                ref="more"
                class="right-side-item lg:!hidden flex"
                :class="{ active: popupVisible.more }"
              >
                <a title="More">
                  <div i-mingcute:menu-line />
                </a>

                <Transition name="slide-in">
                  <MorePop v-show="popupVisible.more" class="bew-popover" />
                </Transition>
              </div>

              <!-- Upload -->
              <div
                ref="upload"
                class="upload right-side-item"
              >
                <a
                  href="https://member.bilibili.com/platform/upload/video/frame"
                  target="_blank"
                  :title="$t('topbar.upload')"
                  bg="$bew-theme-color"
                  rounded-40px
                  un-text="!white !base"
                  w-35px h-35px ml-1
                  flex="~ justify-center"
                  shadow
                  filter="hover:brightness-110"
                  style="--un-shadow: 0 0 10px var(--bew-theme-color-60)"
                >
                  <div i-mingcute:upload-2-line flex-shrink-0 />
                <!-- <span m="l-2" class="hidden xl:block">{{
                  $t('topbar.upload')
                }}</span> -->
                </a>

                <Transition name="slide-in">
                  <UploadPop
                    v-if="popupVisible.upload"
                    class="bew-popover"
                    pos="!left-auto !right-0"
                    transform="!translate-x-0"
                  />
                </Transition>
              </div>
            </template>
          </div>
        </div>
      </main>
    </header>
  </Transition>
</template>

<style lang="scss" scoped>
.top-bar-enter-active,
.top-bar-leave-active {
  transition: all 0.5s ease;
}

.top-bar-enter-from,
.top-bar-leave-to {
  --uno: "opacity-0 transform -translate-y-full";
}

.slide-in-enter-active,
.slide-in-leave-active {
  --uno: "transition-all duration-300 pointer-events-none transform-gpu";
}

.slide-in-leave-to,
.slide-in-enter-from {
  --uno: "transform important: translate-y-4 opacity-0";
}

.slide-out-enter-active,
.slide-out-leave-active {
  --uno: "transition-all duration-300 pointer-events-none transform-gpu";
}

.slide-out-leave-to,
.slide-out-enter-from {
  --uno: "transform important: -translate-y-4 opacity-0";
}

.fade-enter-active,
.fade-leave-active {
  --uno: "transition-all duration-600";
}

.fade-leave-to,
.fade-enter-from {
  --uno: "opacity-0";
}

.hide {
  transform: translateY(-100%);
}

.bew-popover {
  --uno: "absolute top-60px left-1/2";
  --uno: "transform -translate-x-1/2";
  --uno: "overflow-visible";
  --uno: "after:content-empty";
  --uno: "after:opacity-100 after:w-full after:h-100px";
  --uno: "after:absolute after:top--30px after:left-1/2 after:-z-1";
  --uno: "after:transform after:-translate-x-1/2";
}

.logo.activated {
  --uno: "bg-$bew-theme-color dark:bg-white";

  svg {
    --uno: "fill-white dark:fill-$bew-theme-color";
  }
}

.right-side {
  .avatar {
    --uno: "flex items-center mr-4 relative z-1";

    .avatar-img,
    .avatar-shadow {
      --uno: "duration-300";

      &.hover {
        --uno: "transform scale-230 translate-y-36px";
      }
    }

    .avatar-shadow {
      --uno: "opacity-0";

      &.hover {
        --uno: "opacity-60";
      }
    }

    .vip-img {
      &.hover {
        --uno: "transform scale-180 translate-y-55px translate-15px";
      }
    }
  }

  .others.inactive,
  .others:hover {
    --uno: "important-backdrop-filter-none important-bg-$bew-elevated-solid";
  }

  .unread-num-dot {
    --uno: "absolute top-4px right--4px";
    --uno: "important:px-1 rounded-full";
    --uno: "text-xs leading-0 z-6 min-w-16px h-16px";
    --uno: "grid place-items-center";
    --uno: "bg-$bew-theme-color  text-white";
    box-shadow: 0 2px 4px rgba(var(--tw-shadow-color), 0.4);
  }

  .unread-dot {
    --uno: "w-8px h-8px bg-$bew-theme-color rounded-8px absolute right-0 top-6px";
  }

  .right-side-item {
    --uno: "relative text-$bew-text-1 flex items-center";

    &:not(.avatar) a {
      --uno: "text-lg flex items-center p-2 rounded-40px duration-300 relative z-5";
      --uno: "h-35px h-35px";
    }

    &.active a,
    &:not(.upload) a:hover {
      --un-drop-shadow: drop-shadow(0 0 6px white);
      --uno: "bg-$bew-fill-2 shadow-[var(--bew-shadow-edge-glow-1),var(--bew-shadow-1)]";
    }
  }

  .right-side-item .login {
    --un-drop-shadow: drop-shadow(0 0 6px var(--bew-theme-color));
    --uno: "rounded-full mx-1 important:text-$bew-theme-color important:px-4 hover:important-bg-$bew-theme-color hover:important-text-white flex items-center justify-center important:text-base w-120px border-solid border-$bew-theme-color border-2 important:dark:filter";
  }
}
</style>
