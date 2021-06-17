<template>
  <div ref="wrap">
    <div :style="leftSwitch" v-if="navigation" :class="leftSwitchClass" @click="leftSwitchClick">
      <slot name="left-switch"></slot>
    </div>
    <div :style="rightSwitch" v-if="navigation" :class="rightSwitchClass" @click="rightSwitchClick">
      <slot name="right-switch"></slot>
    </div>
    <div
      ref="realBox"
      :style="pos"
      @mouseenter="enter"
      @mouseleave="leave"
      @touchstart="touchStart"
      @touchmove="touchMove"
      @touchend="touchEnd"
    >
      <div ref="slotList" :style="float">
        <slot></slot>
      </div>
      <div v-html="status.copyHtml" :style="float"></div>
    </div>
  </div>
</template>

<script setup>
import {
  computed,
  defineEmit,
  defineProps,
  nextTick,
  onBeforeUnmount,
  onMounted,
  reactive,
  ref,
  watch,
} from '@vue/runtime-core'
import animationFrame from 'comutils/animationFrame'
import copyObj from 'comutils/copyObj'
import arrayEqual from 'comutils/arrayEqual'
const props = defineProps({
  classOption: {
    type: Object,
  },
  data: {
    type: Array,
  },
})
const emit = defineEmit(['ScrollEnd'])

const wrap = ref(null)
const slotList = ref(null)
const realBox = ref(null)
const status = reactive({
  xPos: 0,
  yPos: 0,
  delay: 0,
  copyHtml: '',
  height: 0,
  width: 0, // 外容器宽度
  realBoxWidth: 0, // 内容实际宽度,
  copyHtml: '',
  ease: '',
  realBoxHeight: '',
  reqFrame: null, // move动画的animationFrame定时器
  singleWaitTime: null, // single 单步滚动的定时器
  isHover: false, // mouseenter mouseleave 控制this._move()的开关
  startPos: {},
  startPosY: 0,
  startPosX: 0,
  endPos: {},
})

const defaultOption = computed(() => {
  return {
    step: 1, //步长
    limitMoveNum: 5, //启动无缝滚动最小数据数
    hoverStop: true, //是否启用鼠标hover控制
    direction: 1, // 0 往下 1 往上 2向左 3向右
    openTouch: true, //开启移动端touch
    singleHeight: 0, //单条数据高度有值hoverStop关闭
    singleWidth: 0, //单条数据宽度有值hoverStop关闭
    waitTime: 1000, //单步停止等待时间
    switchOffset: 30,
    autoPlay: true,
    navigation: false,
    switchSingleStep: 134,
    switchDelay: 400,
    switchDisabledClass: 'disabled',
    isSingleRemUnit: false, // singleWidth/singleHeight 是否开启rem度量
  }
})
const options = computed(() => copyObj({}, defaultOption.value, props.classOption))
const navigation = computed(() => options.value.navigation)
const float = computed(() => (isHorizontal ? { float: 'left', overflow: 'hidden' } : { overflow: 'hidden' }))
const isHorizontal = computed(() => options.value.direction > 1)
const leftSwitchState = computed(() => status.xPos < 0)
const autoPlay = computed(() => {
  if (navigation.value) return false
  return options.value.autoPlay
})
const scrollSwitch = computed(() => props.data.length >= options.value.limitMoveNum)
const pos = computed(() => {
  return {
    transform: `translate(${status.xPos}px,${status.yPos}px)`,
    transition: `all ${status.ease} ${status.delay}ms`,
    overflow: 'hidden',
  }
})
const hoverStopSwitch = computed(() => options.value.hoverStop && autoPlay.value && scrollSwitch.value)
const canTouchScroll = computed(() => options.value.openTouch)
const baseFontSize = computed(() =>
  options.value.isSingleRemUnit ? parseInt(window.getComputedStyle(document.documentElement, null).fontSize) : 1
)
const realSingleStopWidth = computed(() => options.value.singleWidth * baseFontSize.value)
const realSingleStopHeight = computed(() => options.value.singleHeight * baseFontSize.value)
const rightSwitchState = computed(() => Math.abs(status.xPos) < status.realBoxWidth - status.width)
const leftSwitchClass = computed(() => (leftSwitchState.value ? '' : options.value.switchDisabledClass))
const rightSwitchClass = computed(() => (rightSwitchState.value ? '' : options.value.switchDisabledClass))
const leftSwitch = computed(() => {
  return {
    position: 'absolute',
    margin: `${status.height / 2}px 0 0 -${options.value.switchOffset}px`,
    transform: 'translate(-100%,-50%)',
  }
})
const rightSwitch = computed(() => {
  return {
    position: 'absolute',
    margin: `${status.height / 2}px 0 0 ${status.width + options.value.switchOffset}px`,
    transform: 'translateY(-50%)',
  }
})
const step = computed(() => {
  let singleStep
  let step = options.value.step
  if (isHorizontal.value) {
    singleStep = realSingleStopWidth.value
  } else {
    singleStep = realSingleStopHeight.value
  }
  if (singleStep > 0 && singleStep % step > 0) {
    console.error('如果设置了单步滚动,step需是单步大小的约数,否则无法保证单步滚动结束的位置是否准确。~~~~~')
  }
  return step
})

const initMove = () => {
  nextTick(() => {
    const switchDelay = options.value.switchDelay
    // const {autoPlay} = autoPlay.value
    // const isHorizontal = isHorizontal.value
    dataWarm(props.data)
    status.copyHtml = '' //清空copy
    if (isHorizontal.value) {
      status.height = wrap.value.offsetHeight
      status.width = wrap.value.offsetWidth
      let slotListWidth = slotList.value.offsetWidth
      // 水平滚动设置warp width
      if (autoPlay.value) {
        // 修正offsetWidth四舍五入
        slotListWidth = slotListWidth * 2 + 1
      }
      realBox.value.style.width = slotListWidth + 'px'
      status.realBoxWidth = slotListWidth
    }
    if (autoPlay.value) {
      status.ease = 'ease-in'
      status.delay = 0
    } else {
      status.ease = 'linear'
      status.delay = switchDelay
      return
    }

    // 是否可以滚动判断
    if (scrollSwitch.value) {
      let timer
      if (timer) clearTimeout(timer)
      status.copyHtml = slotList.value.innerHTML
      setTimeout(() => {
        status.realBoxHeight = realBox.value.offsetHeight
        move()
      }, 0)
    } else {
      cancle()
      status.yPos = status.xPos = 0
    }
  })
}

const dataWarm = (data) => {
  if (data.length > 100) {
    console.warn(`数据达到了${data.length}条有点多哦~,可能会造成部分老旧浏览器卡顿。`)
  }
}
const cancle = () => {
  cancelAnimationFrame(status.reqFrame || '')
}

const leftSwitchClick = () => {
  if (!leftSwitchState.value) return
  // 小于单步距离
  if (Math.abs(status.xPos) < options.value.switchSingleStep) {
    status.xPos = 0
    return
  }
  status.xPos += options.value.switchSingleStep
}
const reset = () => {
  cancle()
  initMove()
}

const rightSwitchClick = () => {
  if (!rightSwitchState.value) return
  // 小于单步距离
  if (status.realBoxWidth - status.width + status.xPos < options.value.switchSingleStep) {
    status.xPos = status.width - status.realBoxWidth
    return
  }
  status.xPos -= options.value.switchSingleStep
}

const startMove = () => {
  status.isHover = false //开启_move
  move()
}

const move = () => {
  // 鼠标移入时拦截_move()
  if (status.isHover) return
  cancle() //进入move立即先清除动画 防止频繁touchMove导致多动画同时进行
  status.reqFrame = requestAnimationFrame(function () {
    const h = status.realBoxHeight / 2 //实际高度
    const w = status.realBoxWidth / 2 //宽度
    let { direction, waitTime } = options.value
    if (direction === 1) {
      // 上
      if (Math.abs(status.yPos) >= h) {
        emit('ScrollEnd')
        console.dir(emit)
        status.yPos = 0
      }
      status.yPos -= step.value
    } else if (direction === 0) {
      // 下
      if (status.yPos >= 0) {
        emit('ScrollEnd')
        status.yPos = h * -1
      }
      status.yPos += step.value
    } else if (direction === 2) {
      // 左
      if (Math.abs(status.xPos) >= w) {
        emit('ScrollEnd')
        status.xPos = 0
      }
      status.xPos -= step.value
    } else if (direction === 3) {
      // 右
      if (status.xPos >= 0) {
        emit('ScrollEnd')
        status.xPos = w * -1
      }
      status.xPos += step.value
    }
    if (status.singleWaitTime) clearTimeout(status.singleWaitTime)
    if (!!realSingleStopHeight.value) {
      //是否启动了单行暂停配置
      if (Math.abs(status.yPos) % realSingleStopHeight.value < step.value) {
        // 符合条件暂停waitTime
        status.singleWaitTime = setTimeout(() => {
          move()
        }, waitTime)
      } else {
        move()
      }
    } else if (!!realSingleStopWidth.value) {
      if (Math.abs(status.xPos) % realSingleStopWidth.value < step.value) {
        // 符合条件暂停waitTime
        status.singleWaitTime = setTimeout(() => {
          move()
        }, waitTime)
      } else {
        move()
      }
    } else {
      move()
    }
  })
}

const stopMove = () => {
  status.isHover = true //关闭_move
  // 防止频频hover进出单步滚动,导致定时器乱掉
  if (status.singleWaitTime) clearTimeout(status.singleWaitTime)
  cancle()
}

const touchStart = (e) => {
  if (!canTouchScroll.value) return
  let timer
  const touch = e.targetTouches[0] //touches数组对象获得屏幕上所有的touch，取第一个touch
  // const { waitTime, singleHeight, singleWidth } = this.options
  status.startPos = {
    x: touch.pageX,
    y: touch.pageY,
  } //取第一个touch的坐标值
  status.startPosY = status.yPos //记录touchStart时候的posY
  status.startPosX = status.xPos //记录touchStart时候的posX
  if (!!options.value.singleHeight && !!options.value.singleWidth) {
    if (timer) clearTimeout(timer)
    timer = setTimeout(() => {
      cancle()
    }, options.value.waitTime + 20)
  } else {
    cancle()
  }
}

const enter = () => {
  if (hoverStopSwitch.value) stopMove()
}
const leave = () => {
  if (hoverStopSwitch.value) startMove()
}

const touchMove = (e) => {
  //当屏幕有多个touch或者页面被缩放过，就不执行move操作
  if (!canTouchScroll.value || e.targetTouches.length > 1 || (e.scale && e.scale !== 1)) return
  const touch = e.targetTouches[0]
  // const { direction } = this.options
  status.endPos = {
    x: touch.pageX - status.startPos.x,
    y: touch.pageY - status.startPos.y,
  }
  e.preventDefault() //阻止触摸事件的默认行为，即阻止滚屏
  const dir = Math.abs(status.endPos.x) < Math.abs(status.endPos.y) ? 1 : 0 //dir，1表示纵向滑动，0为横向滑动
  if (dir === 1 && options.value.direction < 2) {
    // 表示纵向滑动 && 运动方向为上下
    status.yPos = status.startPosY + status.endPos.y
  } else if (dir === 0 && options.value.direction > 1) {
    // 为横向滑动 && 运动方向为左右
    status.xPos = status.startPosX + status.endPos.x
  }
}

const touchEnd = () => {
  if (!canTouchScroll.value) return
  let timer
  const direction = options.value.direction
  status.delay = 50
  if (direction === 1) {
    if (status.yPos > 0) status.yPos = 0
  } else if (direction === 0) {
    let h = (status.realBoxHeight / 2) * -1
    if (status.yPos < h) status.yPos = h
  } else if (direction === 2) {
    if (status.xPos > 0) status.xPos = 0
  } else if (direction === 3) {
    let w = status.realBoxWidth * -1
    if (status.xPos < w) status.xPos = w
  }
  if (timer) clearTimeout(timer)
  timer = setTimeout(() => {
    status.delay = 0
    move()
  }, status.delay)
}

watch(
  () => props.data,
  (newVal, oldVal) => {
    dataWarm(newVal)
    if (!arrayEqual(newVal, oldVal)) {
      reset()
    }
  },
  () => autoPlay,
  (bol) => {
    if (bol) {
      reset()
    } else {
      stopMove()
    }
  }
)
onMounted(() => {
  animationFrame()

  initMove()
})

onBeforeUnmount(() => {
  cancle()
  clearTimeout(status.singleWaitTime)
})
</script>

<style scoped></style>
