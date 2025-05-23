<script setup lang="ts">
import { useI18n } from 'vue-i18n'

import type { FavoriteCategory, FavoriteResource } from '~/components/TopBar/types'
import { useBewlyApp } from '~/composables/useAppProvider'
import { TOP_BAR_VISIBILITY_CHANGE } from '~/constants/globalEvents'
import { settings } from '~/logic'
import type { FavoritesResult, Media as FavoriteItem } from '~/models/video/favorite'
import type { FavoritesCategoryResult, List as CategoryItem } from '~/models/video/favoriteCategory'
import api from '~/utils/api'
import { getCSRF, getUserID, openLinkToNewTab, removeHttpFromUrl } from '~/utils/main'
import emitter from '~/utils/mitt'

const { t } = useI18n()

const favoriteCategories = reactive<CategoryItem[]>([])
const favoriteResources = reactive<FavoriteItem[]>([])
const categoryOptions = reactive<Array<{ value: any, label: string }>>([])

const selectedCategory = ref<FavoriteCategory>()
const activatedCategoryCover = ref<string>('')

const shouldMoveCtrlBarUp = ref<boolean>(false)
const currentPageNum = ref<number>(1)
const keyword = ref<string>('')
const { handlePageRefresh, handleReachBottom, haveScrollbar } = useBewlyApp()
const isLoading = ref<boolean>(false)
const isFullPageLoading = ref<boolean>(false)
const noMoreContent = ref<boolean>(false)

onMounted(() => {
  emitter.off(TOP_BAR_VISIBILITY_CHANGE)
  emitter.on(TOP_BAR_VISIBILITY_CHANGE, (val) => {
    shouldMoveCtrlBarUp.value = false

    // Allow moving tabs up only when the top bar is not hidden & is set to auto-hide
    // This feature is primarily designed to compatible with the Bilibili Evolved's top bar
    // Even when the BewlyBewly top bar is hidden, the Bilibili Evolved top bar still exists, so not moving up
    if (settings.value.autoHideTopBar && settings.value.showTopBar) {
      if (val)
        shouldMoveCtrlBarUp.value = false

      else
        shouldMoveCtrlBarUp.value = true
    }
  })

  initPageAction()
  initData()
})

async function initData() {
  await getFavoriteCategories()
  changeCategory(favoriteCategories[0])
}

onUnmounted(() => {
  emitter.off(TOP_BAR_VISIBILITY_CHANGE)
})

function initPageAction() {
  handleReachBottom.value = async () => {
    if (isLoading.value)
      return
    if (noMoreContent.value)
      return

    await getFavoriteResources(selectedCategory.value!.id, ++currentPageNum.value, keyword.value)
  }

  handlePageRefresh.value = () => {
    if (isLoading.value)
      return
    favoriteResources.length = 0
    currentPageNum.value = 1
    noMoreContent.value = false
    handleSearch()
  }
}

async function getFavoriteCategories() {
  await api.favorite.getFavoriteCategories({
    up_mid: getUserID(),
  })
    .then((res: FavoritesCategoryResult) => {
      if (res.code === 0) {
        Object.assign(favoriteCategories, res.data.list)

        categoryOptions.length = 0
        favoriteCategories.forEach((item) => {
          categoryOptions.push({
            label: item.title,
            value: item,
          })
        })
      }
    })
}

/**
 * Get favorite video resources
 * @param media_id
 * @param pn
 * @param keyword
 */
async function getFavoriteResources(
  media_id: number,
  pn: number,
  keyword = '' as string,
) {
  // if (pn === 1)
  //   isFullPageLoading.value = true
  isLoading.value = true
  try {
    const res: FavoritesResult = await api.favorite.getFavoriteResources({
      media_id,
      pn,
      keyword,
    })

    if (res.code === 0) {
      activatedCategoryCover.value = res.data.info.cover

      if (Array.isArray(res.data.medias) && res.data.medias.length > 0)
        favoriteResources.push(...res.data.medias)

      if (!res.data.medias)
        noMoreContent.value = true

      if (!haveScrollbar() && !noMoreContent.value)
        await getFavoriteResources(selectedCategory.value!.id, ++currentPageNum.value, keyword)
    }
  }
  finally {
    isLoading.value = false
    // isFullPageLoading.value = false
  }
}

async function changeCategory(categoryItem: FavoriteCategory) {
  if (isLoading.value)
    return
  currentPageNum.value = 1
  selectedCategory.value = categoryItem
  favoriteResources.length = 0
  noMoreContent.value = false

  getFavoriteResources(categoryItem.id, 1)
}

function handleSearch() {
  currentPageNum.value = 1
  favoriteResources.length = 0
  noMoreContent.value = false

  getFavoriteResources(selectedCategory.value!.id, currentPageNum.value, keyword.value)
}

function handlePlayAll() {
  openLinkToNewTab(`https://www.bilibili.com/list/ml${selectedCategory.value?.id}`)
}

function jumpToLoginPage() {
  location.href = 'https://passport.bilibili.com/login'
}

function handleUnfavorite(favoriteResource: FavoriteResource) {
  const result = confirm(
    t('favorites.unfavorite_confirm'),
  )
  if (result) {
    api.favorite.patchDelFavoriteResources({
      resources: `${favoriteResource.id}:${favoriteResource.type}`,
      media_id: selectedCategory.value?.id,
      csrf: getCSRF(),
    }).then((res) => {
      if (res.code === 0)
        favoriteResources.splice(favoriteResources.indexOf(favoriteResource as FavoriteItem), 1)
    })
  }
}

function isMusic(item: FavoriteResource) {
  return item.link.includes('bilibili://music')
}
</script>

<template>
  <div v-if="getCSRF()" flex="~ col md:row lg:row" gap-6>
    <main w="full md:60% lg:70% xl:75%" order="2 md:1 lg:1" relative>
      <!-- <h3 text="3xl $bew-text-1" font-bold mb-6>
        {{ selectedCategory?.title }}
      </h3> -->
      <div
        fixed z-10 absolute p-2 flex="~ gap-2"
        items-center
        bg="$bew-elevated-solid" rounded="$bew-radius" shadow="$bew-shadow-2" mt--2 transition="all 300 ease-in-out"
        :class="{ hide: shouldMoveCtrlBarUp }"
      >
        <Select v-model="selectedCategory" w-150px :options="categoryOptions" @change="(val: FavoriteCategory) => changeCategory(val)" />
        <Input v-model="keyword" w-250px @enter="handleSearch" />
        <Button type="primary" @click="handleSearch">
          <template #left>
            <div i-tabler:search />
          </template>
        </Button>
        <!-- <h3
          text="2xl $bew-text-1" font-bold w-150px
        >
          {{ selectedCategory?.title }}
        </h3> -->
      </div>
      <Empty v-if="favoriteResources.length === 0 && !isLoading" m="t-55px b-6" />
      <template v-else>
        <Transition name="fade">
          <Loading v-if="isFullPageLoading" w-full h-full pos="absolute top-0 left-0" mt--50px />
        </Transition>
        <!-- favorite list -->
        <div grid="~ 2xl:cols-4 xl:cols-3 lg:cols-2 md:cols-1 sm:cols-1 cols-1 gap-5" m="t-55px b-6">
          <TransitionGroup name="list">
            <VideoCard
              v-for="item in favoriteResources"
              :key="item.id"
              :video="{
                id: item.id,
                duration: item.duration,
                title: item.title,
                cover: item.cover,
                author: {
                  name: item.upper.name,
                  authorFace: item.upper.face,
                  mid: item.upper.mid,
                },
                view: item.cnt_info.play,
                danmaku: item.cnt_info.danmaku,
                publishedTimestamp: item.pubtime,
                bvid: isMusic(item) ? undefined : item.bvid,
                url: isMusic(item) ? `https://www.bilibili.com/audio/au${item.id}` : undefined,
              }"
              group
            >
              <template #coverTopLeft>
                <button
                  p="x-2 y-1" m="1"
                  rounded="$bew-radius"
                  text="!white xl"
                  bg="black opacity-60 hover:$bew-error-color-80"
                  @click.prevent.stop="handleUnfavorite(item)"
                >
                  <Tooltip :content="$t('favorites.unfavorite')" placement="bottom" type="dark">
                    <div i-ic-baseline-clear />
                  </Tooltip>
                </button>
              </template>
            </VideoCard>
          </TransitionGroup>
        </div>

        <!-- no more content -->
        <Empty v-if="noMoreContent" class="py-4" :description="$t('common.no_more_content')" />

        <!-- loading -->
        <Transition name="fade">
          <Loading
            v-if="isLoading && favoriteResources.length !== 0 && !noMoreContent"
            m="-t-4"
          />
        </Transition>
      </template>
    </main>

    <aside relative w="full md:40% lg:30% xl:25%" class="hidden md:block" order="1 md:2 lg:2">
      <div
        pos="sticky top-120px"
        w-full h="auto md:[calc(100vh-160px)]"
        my-10
        rounded="$bew-radius"
        overflow-hidden
      >
        <!-- Frosted Glass Cover -->
        <div
          pos="absolute top-0 left-0" w-full h-full
          z--1
        >
          <div
            absolute w-full h-full
            bg="$bew-fill-4"
          />
          <img
            v-if="activatedCategoryCover"
            :src="removeHttpFromUrl(`${activatedCategoryCover}@480w_270h_1c`)"
            w-full h-full object="cover center" blur-40px
            relative z--1
          >
        </div>

        <!-- Content -->
        <main
          pos="absolute top-0 left-0"
          w-full h-full
          overflow-overlay
          flex="~ col gap-4 justify-start"
          p-6
        >
          <picture
            rounded="$bew-radius" style="box-shadow: 0 16px 24px -12px rgba(0, 0, 0, .36)"
            aspect-video mb-4 bg="$bew-skeleton"
          >
            <img
              v-if="activatedCategoryCover" :src="removeHttpFromUrl(`${activatedCategoryCover}@480w_270h_1c`)"
              rounded="$bew-radius" aspect-video w-full object-cover
            >
            <div v-else aspect-video w-full>
              <!-- <Empty /> -->
            </div>
          </picture>

          <h3 text="3xl white" fw-600 style="text-shadow: 0 0 12px rgba(0,0,0,.3)">
            {{ selectedCategory?.title }}
          </h3>
          <p flex="~ col" gap-4>
            <Button
              color="rgba(255,255,255,.35)" block text-color="white" strong flex-1
              @click="handlePlayAll"
            >
              <template #left>
                <div i-tabler:player-play />
              </template>
              {{ t('common.play_all') }}
            </Button>
          </p>
          <ul
            class="category-list" h-full min-h-200px
            overflow-overlay
            border="1 color-[rgba(255,255,255,.2)]"
            rounded="$bew-radius"
          >
            <li
              v-for="item in favoriteCategories" :key="item.id"
              border-b="1 color-[rgba(255,255,255,.2)]"
              lh-30px px-4 cursor-pointer hover:bg="[rgba(255,255,255,.35)]"
              duration-300 color-white flex justify-between
              :style="{ background: item.id === selectedCategory?.id ? 'rgba(255,255,255,.35)' : '', pointerEvents: isFullPageLoading ? 'none' : 'auto' }"
              @click="changeCategory(item)"
            >
              <span>{{ item.title }}</span> <span ml-2 color-white color-opacity-60>{{ item.media_count }}</span>
            </li>
          </ul>
        </main>
      </div>
    </aside>
  </div>
  <Empty v-else mt-6 :description="t('common.please_log_in_first')">
    <Button type="primary" @click="jumpToLoginPage()">
      {{ $t('common.login') }}
    </Button>
  </Empty>
</template>

<style lang="scss" scoped>
.hide {
  transform: translateY(-70px);
}

.category-list {
  &::-webkit-scrollbar {
    width: 8px;
    height: 8px;
  }

  &::-webkit-scrollbar-track {
    background: transparent;
  }

  &::-webkit-scrollbar-thumb {
    background-color: rgba(255, 255, 255, 0.35);
    border-radius: 20px;
  }

  &::-webkit-scrollbar-corner {
    background: transparent;
  }
}
</style>
