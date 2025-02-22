<template>
  <view :style="customStyle" :class="`wd-drop-menu ${customClass}`" @click.stop="noop">
    <view class="wd-drop-menu__list">
      <view
        v-for="(child, index) in children"
        :key="index"
        @click="toggle(child)"
        :class="`wd-drop-menu__item ${child.disabled ? 'is-disabled' : ''} ${currentUid === child.$.uid ? 'is-active' : ''}`"
      >
        <view class="wd-drop-menu__item-title">
          <view class="wd-drop-menu__item-title-text">{{ getDisplayTitle(child) }}</view>
          <wd-icon name="arrow-down" size="14px" custom-class="wd-drop-menu__arrow" />
        </view>
      </view>
    </view>
    <slot />
  </view>
</template>
<script lang="ts">
export default {
  name: 'wd-drop-menu',
  options: {
    virtualHost: true,
    addGlobalClass: true,
    styleIsolation: 'shared'
  }
}
</script>

<script lang="ts" setup>
import { getCurrentInstance, inject, onBeforeMount, ref, watch } from 'vue'
import { closeOther } from '../common/clickoutside'
import { type Queue, queueKey } from '../composables/useQueue'
import { getRect } from '../common/util'
import { useChildren } from '../composables/useChildren'
import { DROP_MENU_KEY } from './types'

type DropDirction = 'up' | 'down'
interface Props {
  customClass?: string
  customStyle?: string
  zIndex?: number
  direction?: DropDirction
  modal?: boolean
  closeOnClickModal?: boolean
  duration?: number
}

const props = withDefaults(defineProps<Props>(), {
  customClass: '',
  customStyle: '',
  zIndex: 12,
  direction: 'down',
  modal: true,
  closeOnClickModal: true,
  duration: 200
})
const queue = inject<Queue | null>(queueKey, null)

// -1表示折叠
const currentUid = ref<number | null>(null)
const offset = ref<number>(0)
const windowHeight = ref<number>(0)

const { proxy } = getCurrentInstance() as any

const { linkChildren, children } = useChildren(DROP_MENU_KEY)

linkChildren({ props, fold, offset })

watch(
  () => props.direction,
  (newValue) => {
    if (newValue !== 'up' && newValue !== 'down') {
      // eslint-disable-next-line quotes
      console.error("[wot design] warning(wd-drop-menu): direction must be 'up' or 'down'")
    }
  },
  { deep: true, immediate: true }
)

onBeforeMount(() => {
  windowHeight.value = uni.getSystemInfoSync().windowHeight
})

function noop(event: Event) {
  event.preventDefault()
  event.stopPropagation()
}

function getDisplayTitle(child: any) {
  const { title, modelValue, options, valueKey, labelKey } = child

  if (title) {
    return title
  }
  for (let i = 0, len = options.length; i < len; i++) {
    if (modelValue === options[i][valueKey]) {
      return options[i][labelKey]
    }
  }
  console.error('[wot-design] warning(wd-drop-menu-item): no value is matched in the options option.')
}

function toggle(child: any) {
  // 点击当前 menu, 关闭其他 menu

  if (child && !child.disabled) {
    if (queue && queue.closeOther) {
      queue.closeOther(child)
    } else {
      closeOther(child)
    }
    fold(child)
  }
}
/**
 * 控制菜单内容是否展开
 */
function fold(child?: any) {
  currentUid.value = child ? child.$.uid : null
  if (!child) {
    children.forEach((item) => {
      item.$.exposed!.setShowPop(false)
    })
    return
  }
  getRect('.wd-drop-menu', false, proxy).then((rect: any) => {
    if (!rect) return
    const { top, bottom } = rect

    if (props.direction === 'down') {
      offset.value = bottom
    } else {
      offset.value = windowHeight.value - top
    }
    const showPop = child.$.exposed!.getShowPop()
    if (showPop) {
      child.$.exposed!.setShowPop(false)
      currentUid.value = null
    } else {
      // 选中当前关掉其他的
      children.forEach((item) => {
        if (child.$.uid === item.$.uid) {
          item.$.exposed!.open()
        } else {
          item.$.exposed!.setShowPop(false)
        }
      })
    }
  })
}
</script>

<style lang="scss" scoped>
@import './index.scss';
</style>
