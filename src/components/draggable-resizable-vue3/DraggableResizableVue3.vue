<template>
  <div
    ref="el"
    :style="style"
    :class="[
      {
        [classNameActive]: active,
        [classNameDragging]: dragging,
        [classNameResizing]: resizing,
        [classNameDraggable]: draggable,
        [classNameResizable]: resizable,
      },
      className,
    ]"
    @mousedown="elementMouseDown"
    @touchstart="elementTouchDown"
    @mouseover="elementMouseOver"
    @mouseleave="elementMouseLeave"
  >
    <div :class="'drv-' + handlesType">
      <div
        v-for="handleEl in actualHandles"
        :key="handleEl"
        :class="[
          classNameHandle,
          classNameHandle + '-' + handleEl,
          classNameHandle !== 'drv-handle' ? 'drv-handle' : '',
          classNameHandle !== 'drv-handle' ? 'drv-handle' + '-' + handleEl : '',
        ]"
        :style="{
          display: active ? 'block' : 'none',
        }"
        @mousedown.stop.prevent="handleDown(handleEl, $event)"
        @touchstart.stop.prevent="handleTouchDown(handleEl, $event)"
      >
        <slot
          v-if="slots['handle-' + handleEl]"
          :name="'handle-' + handleEl"
        ></slot>
        <slot v-if="!slots['handle-' + handleEl]" name="handle"></slot>
      </div>
    </div>
    <slot></slot>
  </div>
</template>

<script setup>
import {
  matchesSelectorToParentElements,
  getComputedSize,
  addEvent,
  removeEvent,
} from './utils/dom'
import {
  computeWidth,
  computeHeight,
  restrictToBounds,
  snapToGrid,
} from './utils/fns'
import { ref, computed, onMounted, watch, useSlots, inject, onUnmounted } from 'vue'

const events = {
  mouse: {
    start: 'mousedown',
    move: 'mousemove',
    stop: 'mouseup',
  },

  touch: {
    start: 'touchstart',
    move: 'touchmove',
    stop: 'touchend',
  },
}

const userSelectNone = {
  userSelect: 'none',
  MozUserSelect: 'none',
  WebkitUserSelect: 'none',
  MsUserSelect: 'none',
}

const userSelectAuto = {
  userSelect: 'auto',
  MozUserSelect: 'auto',
  WebkitUserSelect: 'auto',
  MsUserSelect: 'auto',
}

let eventsFor = events.mouse

const props = defineProps({
  draggable: {
    type: Boolean,
    default: true,
  },
  resizable: {
    type: Boolean,
    default: true,
  },
  x: {
    type: Number,
    default: 0,
  },
  y: {
    type: Number,
    default: 0,
  },
  z: {
    type: [String, Number],
    default: 'auto',
    validator: (val) => (typeof val === 'string' ? val === 'auto' : val >= 0),
  },
  w: {
    type: [Number, String],
    // default: 200,
    validator: (val) => {
      if (typeof val === 'number') {
        return val > 0
      }

      return val === 'auto'
    },
  },
  h: {
    type: [Number, String],
    // default: 200,
    validator: (val) => {
      if (typeof val === 'number') {
        return val > 0
      }
      return val === 'auto'
    },
  },
  minWidth: {
    type: Number,
    default: 0,
    validator: (val) => val >= 0,
  },
  minHeight: {
    type: Number,
    default: 0,
    validator: (val) => val >= 0,
  },
  maxWidth: {
    type: Number,
    default: null,
    validator: (val) => val >= 0,
  },
  maxHeight: {
    type: Number,
    default: null,
    validator: (val) => val >= 0,
  },
  active: {
    type: Boolean,
    default: false,
  },
  preventDeactivation: {
    type: Boolean,
    default: false,
  },
  activeOnHover: {
    type: Boolean,
    default: false,
  },
  disableUserSelect: {
    type: Boolean,
    default: true,
  },
  enableNativeDrag: {
    type: Boolean,
    default: false,
  },
  lockAspectRatio: {
    type: Boolean,
    default: false,
  },
  handlesType: {
    type: String,
    default: 'handles',
    validator: (val) => ['handles', 'borders', 'custom'].includes(val),
  },
  handles: {
    type: Array,
    default: () => ['tl', 'tm', 'tr', 'mr', 'br', 'bm', 'bl', 'ml'],
    validator: (val) => {
      const s = new Set(['tl', 'tm', 'tr', 'mr', 'br', 'bm', 'bl', 'ml'])
      return new Set(val.filter((h) => s.has(h))).size === val.length
    },
  },
  handlesSize: {
    type: Number,
    default: 10,
  },
  handlesBorder: {
    type: String,
    default: '0.5px solid #bbb',
  },
  dragHandle: {
    type: String,
    default: null,
  },
  dragCancel: {
    type: String,
    default: null,
  },
  axis: {
    type: String,
    default: 'both',
    validator: (val) => ['x', 'y', 'both'].includes(val),
  },
  grid: {
    type: Array,
    // default: () => [1, 1],
    validator: (val) =>
      Array.isArray(val) &&
      typeof val[0] === 'number' &&
      typeof val[1] === 'number',
  },
  showGrid: {
    type: [Boolean, String],
    default: false,
    validator: (val) => [true, false, 'x', 'y', 'both'].includes(val),
  },
  gridColor: {
    type: String,
    default: 'rgba(0, 0, 0, 0.1)',
  },
  parent: {
    type: [Boolean, String],
    default: false,
  },
  scale: {
    type: [Number, Array],
    default: 1,
    validator: (val) => {
      if (typeof val === 'number') {
        return val > 0
      }

      return val.length === 2 && val[0] > 0 && val[1] > 0
    },
  },
  className: {
    type: String,
    default: 'drv',
  },
  classNameDraggable: {
    type: String,
    default: 'drv-draggable',
  },
  classNameResizable: {
    type: String,
    default: 'drv-resizable',
  },
  classNameDragging: {
    type: String,
    default: 'drv-dragging',
  },
  classNameResizing: {
    type: String,
    default: 'drv-resizing',
  },
  classNameActive: {
    type: String,
    default: 'drv-active',
  },
  classNameHandle: {
    type: String,
    default: 'drv-handle',
  },
  onDragStart: {
    type: Function,
    default: () => true,
  },
  onDrag: {
    type: Function,
    default: () => true,
  },
  onResizeStart: {
    type: Function,
    default: () => true,
  },
  onResize: {
    type: Function,
    default: () => true,
  },
})

const emit = defineEmits([
  'update:x',
  'update:y',
  'update:w',
  'update:h',
  'update:active',
  'resizing',
  'resizestop',
  'dragging',
  'dragstop',
  'activated',
  'deactivated',
])

const el = ref(null)
const parentEl = ref(null)
const noProps = ref({ h: 'auto', w: 'auto', active: false, x: 0, y: 0 })
const slots = useSlots()

const width = computed({
  get() {
    return props.w || noProps.value.w
  },
  set(value) {
    emit('update:w', value)
    noProps.value.w = value
  },
})

const height = computed({
  get() {
    return props.h || noProps.value.h
  },
  set(value) {
    emit('update:h', value)
    noProps.value.h = value
  },
})

const left = computed({
  get() {
    return props.x || noProps.value.x
  },

  set(value) {
    emit('update:x', value)
    noProps.value.x = value
  },
})

const top = computed({
  get() {
    return props.y || noProps.value.y
  },
  set(value) {
    emit('update:y', value)
    noProps.value.y = value
  },
})

const right = ref(null)
const bottom = ref(null)

const active = computed({
  get() {
    return props.active || noProps.value.active
  },
  set(value) {
    emit('update:active', value)
    noProps.value.active = value
  },
})
const handlesSize = computed(() => props.handlesSize + 'px')
const handlesHalfSize = computed(() => props.handlesSize / 2 + 'px')
const handlesBorder = computed(() => props.handlesBorder)

const widthTouched = ref(false)
const heightTouched = ref(false)

const aspectFactor = ref(null)

const parentWidth = ref(null)
const parentHeight = ref(null)

const handle = ref(null)
const resizing = ref(false)
const dragging = ref(false)
const dragEnable = ref(false)
const resizeEnable = ref(false)
const mouseClickPosition = ref(null)
const bounds = ref(null)

const style = computed(() => {
  return {
    transform: `translate(${left.value}px, ${top.value}px)`,
    width: computedWidth.value,
    height: computedHeight.value,
    zIndex: props.z,
    ...(dragging.value && props.disableUserSelect
      ? userSelectNone
      : userSelectAuto),
  }
})

const containerClass = inject('drvContainerClass', {})

const parent = computed(() => {
  return containerClass.value ? '.' + containerClass.value : props.parent
})

const containerGrid = inject('drvContainerGrid', {})

const grid = computed(() => {
  if (!containerGrid || !containerGrid.value) {
    return props.grid || [1, 1]
  }
  return props.grid ? props.grid : containerGrid.value
})

const parentGridBackgroundStyle = computed(() => {
  const axisBg = {
    x: `linear-gradient(-90deg, ${props.gridColor} 1px, transparent 1px) 0px 0px / ${grid.value[0]}px ${grid.value[0]}px`,
    y: `linear-gradient(0deg, ${props.gridColor} 1px, transparent 1px) 0px 0px / ${grid.value[1]}px ${grid.value[1]}px`,
  }

  if (props.showGrid === 'x' || props.showGrid === 'y') {
    return axisBg[props.showGrid]
  }

  return props.showGrid ? `${axisBg.x}, ${axisBg.y}` : null
})

const actualHandles = computed(() => {
  if (!props.resizable) return []

  return props.handles
})

const computedWidth = computed(() => {
  if (width.value === 'auto') {
    if (!widthTouched.value) {
      return 'auto'
    }
  }

  return width.value + 'px'
})

const computedHeight = computed(() => {
  if (height.value === 'auto') {
    if (!heightTouched.value) {
      return 'auto'
    }
  }

  return height.value + 'px'
})
const minW = ref(props.minWidth)
const minH = ref(props.minHeight)
const maxW = ref(props.maxWidth)
const maxH = ref(props.maxHeight)

const resizingOnX = computed(() => {
  return (
    Boolean(handle.value) &&
    (handle.value.includes('l') || handle.value.includes('r'))
  )
})

const resizingOnY = computed(() => {
  return (
    Boolean(handle.value) &&
    (handle.value.includes('t') || handle.value.includes('b'))
  )
})

// const isCornerHandle = computed(() => {
//   return (
//     Boolean(handle.value) && ['tl', 'tr', 'br', 'bl'].includes(handle.value)
//   )
// })

// methods
const resetBoundsAndMouseState = () => {
  mouseClickPosition.value = {
    mouseX: 0,
    mouseY: 0,
    x: 0,
    y: 0,
    w: 0,
    h: 0,
  }

  bounds.value = {
    minLeft: null,
    maxLeft: null,
    minRight: null,
    maxRight: null,
    minTop: null,
    maxTop: null,
    minBottom: null,
    maxBottom: null,
  }
}

const checkParentSize = () => {
  if (parent.value) {
    const [newParentWidth, newParentHeight] = getParentSize()

    parentWidth.value = newParentWidth
    parentHeight.value = newParentHeight

    right.value = parentWidth.value - width.value - left.value
    bottom.value = parentHeight.value - height.value - top.value
  }
}

const getParentSize = () => {
  if (parent.value) {
    const parentElement =
      typeof parent.value === 'string'
        ? el.value.closest(parent.value)
        : el.value.parentNode
    parentEl.value = parentElement
    const style = window.getComputedStyle(parentElement, null)
    return [
      parseInt(style.getPropertyValue('width'), 10),
      parseInt(style.getPropertyValue('height'), 10),
    ]
  }

  return [null, null]
}

const showParentGrid = () => {
  if (parent.value && props.showGrid) {
    if (!parentEl.value) {
      const parent =
        typeof parent.value === 'string'
          ? el.value.closest(parent.value)
          : el.value.parentNode
      parentEl.value = parent
    }

    parentEl.value.style.background = parentGridBackgroundStyle.value
  }
}

const elementTouchDown = (e) => {
  eventsFor = events.touch
  elementDown(e)
}

const elementMouseDown = (e) => {
  eventsFor = events.mouse
  elementDown(e)
}

const elementDown = (e) => {
  if (e instanceof MouseEvent && e.which !== 1) {
    return
  }

  const target = e.target || e.srcElement

  if (el.value.contains(target)) {
    if (props.onDragStart(e) === false) {
      return
    }

    if (
      (props.dragHandle &&
        !matchesSelectorToParentElements(target, props.dragHandle, el.value)) ||
      (props.dragCancel &&
        matchesSelectorToParentElements(target, props.dragCancel, el.value))
    ) {
      dragging.value = false
      return
    }

    if (!active.value) {
      active.value = true
      emit('activated')
      // emit('update:active', true)
    }

    if (props.draggable) {
      dragEnable.value = true
    }

    mouseClickPosition.value.mouseX = e.touches ? e.touches[0].pageX : e.pageX
    mouseClickPosition.value.mouseY = e.touches ? e.touches[0].pageY : e.pageY
    mouseClickPosition.value.left = left.value
    mouseClickPosition.value.right = right.value
    mouseClickPosition.value.top = top.value
    mouseClickPosition.value.bottom = bottom.value

    if (parent.value) {
      bounds.value = calcDragLimits()
    }

    addEvent(document.documentElement, eventsFor.move, move)
    addEvent(document.documentElement, eventsFor.stop, handleUp)
  }
}

const calcDragLimits = () => {
  return {
    minLeft: left.value % grid.value[0],
    maxLeft:
      Math.floor(
        (parentWidth.value - width.value - left.value) / grid.value[0],
      ) *
        grid.value[0] +
      left.value,

    minRight: right.value % grid.value[0],
    maxRight:
      Math.floor(
        (parentWidth.value - width.value - right.value) / grid.value[0],
      ) *
        grid.value[0] +
      right.value,

    minTop: top.value % grid.value[1],
    maxTop:
      Math.floor(
        (parentHeight.value - height.value - top.value) / grid.value[1],
      ) *
        grid.value[1] +
      top.value,
    minBottom: bottom.value % grid.value[1],
    maxBottom:
      Math.floor(
        (parentHeight.value - height.value - bottom.value) / grid.value[1],
      ) *
        grid.value[1] +
      bottom.value,
  }
}

const deselect = (e) => {
  const target = e.target || e.srcElement

  const regex = new RegExp(props.className + '-([trmbl]{2})', '')
  if (
    !el.value ||
    (!el.value.contains(target) && !regex.test(target.className))
  ) {
    if (active.value && !props.preventDeactivation) {
      active.value = false
      emit('deactivated')
      // emit('update:active', false)
    }

    removeEvent(document.documentElement, eventsFor.move, handleResize)
  }

  resetBoundsAndMouseState()
}

const handleTouchDown = (handle, e) => {
  eventsFor = events.touch
  handleDown(handle, e)
}

const handleDown = (handleEl, e) => {
  if (e instanceof MouseEvent && e.which !== 1) {
    return
  }

  if (props.onResizeStart(handleEl, e) === false) {
    return
  }

  if (e.stopPropagation) e.stopPropagation()

  // Here we avoid a dangerous recursion by faking

  // corner handles as middle handles

  if (props.lockAspectRatio && !handleEl.includes('m')) {
    handle.value = 'm' + handleEl.substring(1)
  } else {
    handle.value = handleEl
  }

  resizeEnable.value = true

  mouseClickPosition.value.mouseX = e.touches ? e.touches[0].pageX : e.pageX
  mouseClickPosition.value.mouseY = e.touches ? e.touches[0].pageY : e.pageY
  mouseClickPosition.value.left = left.value
  mouseClickPosition.value.right = right.value
  mouseClickPosition.value.top = top.value
  mouseClickPosition.value.bottom = bottom.value

  bounds.value = calcResizeLimits()

  addEvent(document.documentElement, eventsFor.move, handleResize)
  addEvent(document.documentElement, eventsFor.stop, handleUp)
}

const calcResizeLimits = () => {
  const [gridX, gridY] = grid.value

  if (props.lockAspectRatio) {
    if (minW.value / minH.value > aspectFactor.value) {
      minH.value = minW.value / aspectFactor.value
    } else {
      minW.value = aspectFactor.value * minH.value
    }

    if (maxW.value && maxH.value) {
      maxW.value = Math.min(maxW.value, aspectFactor.value * maxH.value)
      maxH.value = Math.min(maxH.value, maxW.value / aspectFactor.value)
    } else if (maxW.value) {
      maxH.value = maxW.value / aspectFactor.value
    } else if (maxH.value) {
      maxW.value = aspectFactor.value * maxH.value
    }
  }

  maxW.value = maxW.value - (maxW.value % gridX)
  maxH.value = maxH.value - (maxH.value % gridY)

  const limits = {
    minLeft: null,
    maxLeft: null,
    minTop: null,
    maxTop: null,
    minRight: null,
    maxRight: null,
    minBottom: null,
    maxBottom: null,
  }

  if (parent.value) {
    limits.minLeft = left.value % gridX
    limits.maxLeft =
      left.value + Math.floor((width.value - minW.value) / gridX) * gridX
    limits.minTop = top.value % gridY
    limits.maxTop =
      top.value + Math.floor((height.value - minH.value) / gridY) * gridY
    limits.minRight = right.value % gridX
    limits.maxRight =
      right.value + Math.floor((width.value - minW.value) / gridX) * gridX
    limits.minBottom = bottom.value % gridY
    limits.maxBottom =
      bottom.value + Math.floor((height.value - minH.value) / gridY) * gridY

    if (maxW.value) {
      limits.minLeft = Math.max(
        limits.minLeft,
        parentWidth.value - right.value - maxW.value,
      )

      limits.minRight = Math.max(
        limits.minRight,
        parentWidth.value - left.value - maxW.value,
      )
    }

    if (maxH.value) {
      limits.minTop = Math.max(
        limits.minTop,
        parentHeight.value - bottom.value - maxH.value,
      )

      limits.minBottom = Math.max(
        limits.minBottom,
        parentHeight.value - top.value - maxH.value,
      )
    }

    if (props.lockAspectRatio) {
      limits.minLeft = Math.max(
        limits.minLeft,
        left.value - top.value * aspectFactor.value,
      )
      limits.minTop = Math.max(
        limits.minTop,
        top.value - left.value / aspectFactor.value,
      )
      limits.minRight = Math.max(
        limits.minRight,
        right.value - bottom.value * aspectFactor.value,
      )
      limits.minBottom = Math.max(
        limits.minBottom,
        bottom.value - right.value / aspectFactor.value,
      )
    }
  } else {
    limits.minLeft = null
    limits.maxLeft =
      left.value + Math.floor((width.value - minW.value) / gridX) * gridX
    limits.minTop = null
    limits.maxTop =
      top.value + Math.floor((height.value - minH.value) / gridY) * gridY
    limits.minRight = null
    limits.maxRight =
      right.value + Math.floor((width.value - minW.value) / gridX) * gridX
    limits.minBottom = null
    limits.maxBottom =
      bottom.value + Math.floor((height.value - minH.value) / gridY) * gridY

    if (maxW.value) {
      limits.minLeft = -(right.value + maxW.value)
      limits.minRight = -(left.value + maxW.value)
    }

    if (maxH.value) {
      limits.minTop = -(bottom.value + maxH.value)
      limits.minBottom = -(top.value + maxH.value)
    }

    if (props.lockAspectRatio && maxW.value && maxH.value) {
      limits.minLeft = Math.min(limits.minLeft, -(right.value + maxW.value))
      limits.minTop = Math.min(limits.minTop, -(maxH.value + bottom.value))
      limits.minRight = Math.min(limits.minRight, -left.value - maxW.value)
      limits.minBottom = Math.min(limits.minBottom, -top.value - maxH.value)
    }
  }

  return limits
}

const move = (e) => {
  if (resizing.value) {
    handleResize(e)
  } else if (dragEnable.value) {
    handleDrag(e)
  }
}
const handleDrag = (e) => {
  const tmpDeltaX =
    props.axis && props.axis !== 'y'
      ? mouseClickPosition.value.mouseX -
        (e.touches ? e.touches[0].pageX : e.pageX)
      : 0

  const tmpDeltaY =
    props.axis && props.axis !== 'x'
      ? mouseClickPosition.value.mouseY -
        (e.touches ? e.touches[0].pageY : e.pageY)
      : 0

  const [deltaX, deltaY] = snapToGrid(
    grid.value,
    tmpDeltaX,
    tmpDeltaY,
    props.scale,
  )

  const leftPx = restrictToBounds(
    mouseClickPosition.value.left - deltaX,
    bounds.value.minLeft,
    bounds.value.maxLeft,
  )

  const topPx = restrictToBounds(
    mouseClickPosition.value.top - deltaY,
    bounds.value.minTop,
    bounds.value.maxTop,
  )

  if (props.onDrag(left, top) === false) {
    return
  }

  const rightPx = restrictToBounds(
    mouseClickPosition.value.right + deltaX,
    bounds.value.minRight,
    bounds.value.maxRight,
  )

  const bottomPx = restrictToBounds(
    mouseClickPosition.value.bottom + deltaY,
    bounds.value.minBottom,
    bounds.value.maxBottom,
  )

  left.value = leftPx
  top.value = topPx
  right.value = rightPx
  bottom.value = bottomPx

  emit('dragging', left.value, top.value)

  dragging.value = true
}

const moveHorizontally = (val) => {
  // should calculate with scale 1.

  const [deltaX] = snapToGrid(grid.value, val, top.value, 1)

  const leftPx = restrictToBounds(
    deltaX,
    bounds.value.minLeft,
    bounds.value.maxLeft,
  )

  left.value = leftPx
  right.value = parentWidth.value - width.value - leftPx
}

const moveVertically = (val) => {
  // should calculate with scale 1.

  const [, deltaY] = snapToGrid(grid.value, left.value, val, 1)

  const topPx = restrictToBounds(
    deltaY,
    bounds.value.minTop,
    bounds.value.maxTop,
  )

  top.value = topPx
  bottom.value = parentHeight.value - height.value - topPx
}

const handleResize = (e) => {
  let leftPx = left.value
  let topPx = top.value
  let rightPx = right.value
  let bottomPx = bottom.value

  // const lockAspectRatio = props.lockAspectRatio

  const tmpDeltaX =
    mouseClickPosition.value.mouseX - (e.touches ? e.touches[0].pageX : e.pageX)

  const tmpDeltaY =
    mouseClickPosition.value.mouseY - (e.touches ? e.touches[0].pageY : e.pageY)

  if (!widthTouched.value && tmpDeltaX) {
    widthTouched.value = true
  }

  if (!heightTouched.value && tmpDeltaY) {
    heightTouched.value = true
  }

  const [deltaX, deltaY] = snapToGrid(
    grid.value,
    tmpDeltaX,
    tmpDeltaY,
    props.scale,
  )

  if (handle.value.includes('b')) {
    bottomPx = restrictToBounds(
      mouseClickPosition.value.bottom + deltaY,
      bounds.value.minBottom,
      bounds.value.maxBottom,
    )

    if (props.lockAspectRatio && resizingOnY) {
      rightPx = right.value - (bottom.value - bottomPx) * aspectFactor.value
    }
  } else if (handle.value.includes('t')) {
    topPx = restrictToBounds(
      mouseClickPosition.value.top - deltaY,
      bounds.value.minTop,
      bounds.value.maxTop,
    )

    if (props.lockAspectRatio && resizingOnY) {
      leftPx = left.value - (top.value - topPx) * aspectFactor.value
    }
  }

  if (handle.value.includes('r')) {
    rightPx = restrictToBounds(
      mouseClickPosition.value.right + deltaX,
      bounds.value.minRight,
      bounds.value.maxRight,
    )

    if (props.lockAspectRatio && resizingOnX) {
      bottomPx = bottom.value - (right.value - rightPx) / aspectFactor.value
    }
  } else if (handle.value.includes('l')) {
    leftPx = restrictToBounds(
      mouseClickPosition.value.left - deltaX,
      bounds.value.minLeft,
      bounds.value.maxLeft,
    )

    if (props.lockAspectRatio && resizingOnX) {
      topPx = top.value - (left.value - leftPx) / aspectFactor.value
    }
  }

  const widthPx = computeWidth(parentWidth.value, leftPx, rightPx)

  const heightPx = computeHeight(parentHeight.value, topPx, bottomPx)

  if (
    props.onResize(handle.value, leftPx, topPx, widthPx, heightPx) === false
  ) {
    return
  }

  left.value = leftPx
  top.value = topPx
  right.value = rightPx
  bottom.value = bottomPx
  width.value = widthPx
  height.value = heightPx

  emit('resizing', left.value, top.value, width.value, height.value)

  resizing.value = true
}

const changeWidth = (val) => {
  // should calculate with scale 1.

  const [newWidth] = snapToGrid(grid.value, val, 0, 1)

  let rightPx = restrictToBounds(
    parentWidth.value - newWidth - left.value,
    bounds.value.minRight,
    bounds.value.maxRight,
  )

  let bottomPx = bottom.value

  if (props.lockAspectRatio) {
    bottomPx = bottom.value - (right.value - rightPx) / aspectFactor.value
  }

  const widthPx = computeWidth(parentWidth.value, left.value, rightPx)
  const heightPx = computeHeight(parentHeight.value, top.value, bottomPx)

  right.value = rightPx
  bottom.value = bottomPx
  width.value = widthPx
  height.value = heightPx
}

const changeHeight = (val) => {
  // should calculate with scale 1.

  const [, newHeight] = snapToGrid(grid.value, 0, val, 1)

  let bottomPx = restrictToBounds(
    parentHeight.value - newHeight - top.value,
    bounds.value.minBottom,
    bounds.value.maxBottom,
  )

  let rightPx = right.value

  if (props.lockAspectRatio) {
    rightPx = right.value - (bottom.value - bottomPx) * aspectFactor.value
  }

  const widthPx = computeWidth(parentWidth.value, left.value, rightPx)

  const heightPx = computeHeight(parentHeight.value, top.value, bottomPx)

  right.value = rightPx
  bottom.value = bottomPx
  width.value = widthPx
  height.value = heightPx
}

const handleUp = () => {
  handle.value = null

  resetBoundsAndMouseState()

  dragEnable.value = false
  resizeEnable.value = false

  if (resizing.value) {
    resizing.value = false

    emit('resizestop', left.value, top.value, width.value, height.value)
  }

  if (dragging.value) {
    dragging.value = false

    emit('dragstop', left.value, top.value)
  }

  removeEvent(document.documentElement, eventsFor.move, handleResize)
}

const elementMouseOver = () => {
  if (props.activeOnHover) {
    active.value = true
  }
}
const elementMouseLeave = () => {
  if (props.activeOnHover && !dragging.value) {
    active.value = false
  }
}

onMounted(() => {
  resetBoundsAndMouseState()

  if (!props.enableNativeDrag) {
    el.value.ondragstart = () => false
  }

  const [parentWidthPx, parentHeightPx] = getParentSize()

  parentWidth.value = parentWidthPx
  parentHeight.value = parentHeightPx

  const [widthPx, heightPx] = getComputedSize(el.value)

  aspectFactor.value =
    (width.value !== 'auto' ? width.value : widthPx) /
    (height.value !== 'auto' ? height.value : heightPx)

  width.value = width.value !== 'auto' ? width.value : widthPx
  height.value = height.value !== 'auto' ? height.value : heightPx
  right.value = parentWidth.value - width.value - left.value
  bottom.value = parentHeight.value - height.value - top.value

  if (active.value) {
    emit('activated')
  }

  if (props.showGrid) {
    showParentGrid()
  }

  addEvent(document.documentElement, 'mousedown', deselect)
  addEvent(document.documentElement, 'touchend touchcancel', deselect)
  addEvent(window, 'resize', checkParentSize)
})

  
onUnmounted(() => {
  removeEvent(document.documentElement, 'mousedown', deselect)
  removeEvent(document.documentElement, 'touchend touchcancel', deselect)
  removeEvent(window, 'resize', checkParentSize)
})
  
watch(
  () => props.x,
  (val) => {
    if (resizing.value || dragging.value) {
      return
    }

    if (parent.value) {
      bounds.value = calcDragLimits()
    }

    moveHorizontally(val)
  },
)

watch(
  () => props.y,
  (val) => {
    if (resizing.value || dragging.value) {
      return
    }

    if (parent.value) {
      bounds.value = calcDragLimits()
    }

    moveVertically(val)
  },
)

watch(
  () => props.lockAspectRatio,
  (val) => {
    if (val) {
      aspectFactor.value = width.value / height.value
    } else {
      aspectFactor.value = undefined
    }
  },
)

watch(
  () => props.w,
  (val) => {
    if (resizing.value || dragging.value) {
      return
    }

    if (parent.value) {
      bounds.value = calcResizeLimits()
    }

    changeWidth(val)
  },
)

watch(
  () => props.h,
  (val) => {
    if (resizing.value || dragging.value) {
      return
    }

    if (parent.value) {
      bounds.value = calcResizeLimits()
    }

    changeHeight(val)
  },
)

watch(
  () => props.showGrid,
  () => {
    showParentGrid()
  },
)

watch(
  () => props.grid,
  () => {
    showParentGrid()
  },
)
</script>

<style>
.drv {
  touch-action: none;
  position: absolute;
  box-sizing: border-box;
  border: 1px dashed gray;
}

.drv-handles .drv-handle {
  box-sizing: border-box;
  position: absolute;
  min-width: v-bind(handlesSize);
  min-height: v-bind(handlesSize);
  border: v-bind(handlesBorder);
}

.drv-handles .drv-handle-tl {
  transform: translate(-100%, -100%);
  top: 0;
  left: 0;
  cursor: nw-resize;
}

.drv-handles .drv-handle-tm {
  transform: translate(-50%, -100%);
  top: 0;
  left: 50%;
  cursor: n-resize;
}

.drv-handles .drv-handle-tr {
  transform: translate(100%, -100%);
  top: 0;
  right: 0;
  cursor: ne-resize;
}

.drv-handles .drv-handle-ml {
  transform: translate(-100%, -50%);
  top: 50%;
  left: 0;
  cursor: w-resize;
}

.drv-handles .drv-handle-mr {
  transform: translate(100%, -50%);
  top: 50%;
  right: 0;
  cursor: e-resize;
}

.drv-handles .drv-handle-bl {
  transform: translate(-100%, 100%);
  bottom: 0;
  left: 0;
  cursor: sw-resize;
}

.drv-handles .drv-handle-bm {
  transform: translate(-50%, 100%);
  bottom: 0;
  left: 50%;
  cursor: s-resize;
}

.drv-handles .drv-handle-br {
  transform: translate(100%, 100%);
  bottom: 0;
  right: 0;
  cursor: se-resize;
}

.drv-borders .drv-handle {
  box-sizing: border-box;
  position: absolute;
  min-width: v-bind(handlesSize);
  min-height: v-bind(handlesSize);
}

.drv-borders .drv-handle-tl {
  transform: translate(-50%, -50%);
  top: 0;
  left: 0;
  cursor: nw-resize;
}
.drv-borders .drv-handle-tm {
  transform: translate(0, -50%);
  left: v-bind(handlesHalfSize);
  width: calc(100% - v-bind(handlesSize));
  top: 0;
  cursor: n-resize;
}

.drv-borders .drv-handle-tr {
  transform: translate(50%, -50%);
  top: 0;
  right: 0;
  cursor: ne-resize;
}

.drv-borders .drv-handle-ml {
  transform: translate(-50%, 0);
  top: v-bind(handlesHalfSize);
  height: calc(100% - v-bind(handlesSize));
  left: 0;
  cursor: w-resize;
}

.drv-borders .drv-handle-bl {
  transform: translate(-50%, 50%);
  bottom: 0;
  left: 0;
  cursor: sw-resize;
}

.drv-borders .drv-handle-bm {
  transform: translate(0, 50%);
  left: v-bind(handlesHalfSize);
  width: calc(100% - v-bind(handlesSize));
  bottom: 0;
  cursor: s-resize;
}

.drv-borders .drv-handle-br {
  transform: translate(50%, 50%);
  bottom: 0;
  right: 0;
  cursor: se-resize;
}

.drv-borders .drv-handle-mr {
  transform: translate(50%, 0);
  top: v-bind(handlesHalfSize);
  height: calc(100% - v-bind(handlesSize));
  right: 0;
  cursor: e-resize;
}
</style>
