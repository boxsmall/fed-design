<template>
  <div
    ref="wrapper"
    :style="style"
    :class="[{
      [classNameActive]: enabled,
      [classNameDragging]: dragging,
      [classNameResizing]: resizing,
      [classNameDraggable]: draggable,
      [classNameResizable]: resizable
    }, className]"
    @mousedown="elementDown"
    @touchstart="elementTouchDown"
  >
    <!--旋转 +by Adang-->
    <div
      v-if="resizable"
      :class="[classNameHandle, classNameHandle + '-rotate']"
      :style="{display: enabled ? 'block' : 'none'}"
      @mousedown.stop.prevent="handleRotateDown"
    >
      <i class="el-icon-refresh" />
    </div>
    <!--变形-->
    <div
      v-for="item in actualHandles"
      :key="item"
      :class="[classNameHandle, classNameHandle + '-' + item]"
      :style="{display: enabled ? 'block' : 'none'}"
      @mousedown.stop.prevent="handleDown(item, $event)"
      @touchstart.stop.prevent="handleTouchDown(item, $event)"
    >
      <slot :name="item" />
    </div>
    <slot />
  </div>
</template>

<script>
import { matchesSelectorToParentElements, addEvent, removeEvent } from './dom'

const events = {
  mouse: {
    start: 'mousedown',
    move: 'mousemove',
    stop: 'mouseup'
  },
  touch: {
    start: 'touchstart',
    move: 'touchmove',
    stop: 'touchend'
  }
}

const userSelectNone = {
  userSelect: 'none',
  MozUserSelect: 'none',
  WebkitUserSelect: 'none',
  MsUserSelect: 'none'
}

const userSelectAuto = {
  userSelect: 'auto',
  MozUserSelect: 'auto',
  WebkitUserSelect: 'auto',
  MsUserSelect: 'auto'
}

let eventsFor = events.mouse

export default {
  replace: true,
  name: 'XDraggableResizable',
  props: {
    className: {
      type: String,
      default: 'x-draggable-resizable'
    },
    classNameDraggable: {
      type: String,
      default: 'draggable'
    },
    classNameResizable: {
      type: String,
      default: 'resizable'
    },
    classNameDragging: {
      type: String,
      default: 'dragging'
    },
    classNameResizing: {
      type: String,
      default: 'resizing'
    },
    classNameActive: {
      type: String,
      default: 'active'
    },
    classNameHandle: {
      type: String,
      default: 'handle'
    },
    disableUserSelect: {
      type: Boolean,
      default: true
    },
    enableNativeDrag: {
      type: Boolean,
      default: false
    },
    preventDeactivation: {
      type: Boolean,
      default: false
    },
    active: {
      type: Boolean,
      default: false
    },
    draggable: {
      type: Boolean,
      default: true
    },
    resizable: {
      type: Boolean,
      default: true
    },
    lockAspectRatio: {
      type: Boolean,
      default: false
    },
    w: {
      type: Number,
      default: 200,
      validator: (val) => val > 0
    },
    h: {
      type: Number,
      default: 200,
      validator: (val) => val > 0
    },
    minWidth: {
      type: Number,
      default: 0,
      validator: (val) => val >= 0
    },
    minHeight: {
      type: Number,
      default: 0,
      validator: (val) => val >= 0
    },
    maxWidth: {
      type: Number,
      default: null,
      validator: (val) => val >= 0
    },
    maxHeight: {
      type: Number,
      default: null,
      validator: (val) => val >= 0
    },
    x: {
      type: Number,
      default: 0,
      validator: (val) => typeof val === 'number'
    },
    y: {
      type: Number,
      default: 0,
      validator: (val) => typeof val === 'number'
    },
    z: {
      type: [String, Number],
      default: 'auto',
      validator: (val) => (typeof val === 'string' ? val === 'auto' : val >= 0)
    },
    handles: {
      type: Array,
      default: () => ['tl', 'tm', 'tr', 'mr', 'br', 'bm', 'bl', 'ml'],
      validator: (val) => {
        const s = new Set(['tl', 'tm', 'tr', 'mr', 'br', 'bm', 'bl', 'ml'])

        return new Set(val.filter(h => s.has(h))).size === val.length
      }
    },
    dragHandle: {
      type: String,
      default: null
    },
    dragCancel: {
      type: String,
      default: null
    },
    axis: {
      type: String,
      default: 'both',
      validator: (val) => ['x', 'y', 'both'].includes(val)
    },
    grid: {
      type: Array,
      default: () => [1, 1]
    },
    parent: {
      type: Boolean,
      default: false
    },
    scale: {
      type: Number,
      default: 1,
      validator: (val) => val > 0
    },
    onDragStart: {
      type: Function,
      default: null
    },
    onResizeStart: {
      type: Function,
      default: null
    },
    // +by Adang
    // 可取消选中状态 element
    deselectElement: {
      type: String,
      default: null
    },
    // 旋转
    r: {
      type: Number,
      default: 0,
      validator: (val) => typeof val === 'number'
    },
    // 元素对齐
    snap: {
      type: Boolean, default: false
    },
    // 当调用对齐时，用来设置组件与组件之间的对齐距离，以像素为单位
    snapTolerance: {
      type: Number,
      default: 5,
      validator: (val) => typeof val === 'number'
    }
  },

  data () {
    return {
      rawWidth: this.w,
      rawHeight: this.h,
      rawLeft: this.x,
      rawTop: this.y,
      rawRight: null,
      rawBottom: null,

      left: this.x,
      top: this.y,
      right: null,
      bottom: null,

      aspectFactor: this.w / this.h,

      parentWidth: null,
      parentHeight: null,

      minW: this.minWidth,
      minH: this.minHeight,

      maxW: this.maxWidth,
      maxH: this.maxHeight,

      handle: null,
      enabled: this.active,
      resizing: false,
      dragging: false,
      zIndex: this.z,
      // +by Adang
      currentHandle: null,
      rotate: this.r,
      rawRotate: this.r,
      rotateOpt: null,
      bounds: {}
    }
  },
  computed: {
    style () {
      return {
        position: 'absolute',
        top: this.top + 'px',
        left: this.left + 'px',
        width: this.width + 'px',
        height: this.height + 'px',
        zIndex: this.zIndex,
        ...(this.dragging && this.disableUserSelect ? userSelectNone : userSelectAuto),
        // +by Adang
        opacity: this.opacity,
        transform: `rotate(${this.rotate}deg)`
      }
    },
    actualHandles () {
      if (!this.resizable) return []

      return this.handles
    },
    width () {
      return this.parentWidth - this.left - this.right
    },
    height () {
      return this.parentHeight - this.top - this.bottom
    },
    resizingOnX () {
      return (Boolean(this.handle) && (this.handle.includes('l') || this.handle.includes('r')))
    },
    resizingOnY () {
      return (Boolean(this.handle) && (this.handle.includes('t') || this.handle.includes('b')))
    },
    isCornerHandle () {
      return (Boolean(this.currentHandle) && ['tl', 'tr', 'br', 'bl'].includes(this.currentHandle))
    }
  },

  watch: {
    active: {
      handler (val) {
        this.enabled = val
        if (val) {
          this.$emit('activated')
        } else {
          this.$emit('deactivated')
        }
      },
      immediate: true
    },
    z (val) {
      if (val >= 0 || val === 'auto') {
        this.zIndex = val
      }
    },
    // 旋转 +by Adang
    rawRotate (newRotate) {
      this.rotate = newRotate
    },
    rawLeft (newLeft) {
      const bounds = this.bounds
      const aspectFactor = this.aspectFactor
      const lockAspectRatio = this.lockAspectRatio
      const left = this.left
      const top = this.top

      if (bounds.minLeft !== null && newLeft < bounds.minLeft) {
        newLeft = bounds.minLeft
      } else if (bounds.maxLeft !== null && bounds.maxLeft < newLeft) {
        newLeft = bounds.maxLeft
      }

      // +by Adang this.resizingOnX 替换为 this.isCornerHandle
      // 暂时不用
      if (lockAspectRatio && this.resizingOnX) {
        this.rawTop = top - (left - newLeft) / aspectFactor
      }

      this.left = newLeft
    },
    rawRight (newRight) {
      const bounds = this.bounds
      const aspectFactor = this.aspectFactor
      const lockAspectRatio = this.lockAspectRatio
      const right = this.right
      const bottom = this.bottom

      if (bounds.minRight !== null && newRight < bounds.minRight) {
        newRight = bounds.minRight
      } else if (bounds.maxRight !== null && bounds.maxRight < newRight) {
        newRight = bounds.maxRight
      }

      // +by Adang this.resizingOnX 替换为 this.isCornerHandle
      // 暂时不用
      if (lockAspectRatio && this.resizingOnX) {
        this.rawBottom = bottom - (right - newRight) / aspectFactor
      }

      this.right = newRight
    },
    rawTop (newTop) {
      const bounds = this.bounds
      const aspectFactor = this.aspectFactor
      const lockAspectRatio = this.lockAspectRatio
      const left = this.left
      const top = this.top

      if (bounds.minTop !== null && newTop < bounds.minTop) {
        newTop = bounds.minTop
      } else if (bounds.maxTop !== null && bounds.maxTop < newTop) {
        newTop = bounds.maxTop
      }

      if (lockAspectRatio && this.resizingOnY) {
        this.rawLeft = left - (top - newTop) * aspectFactor
      }

      this.top = newTop
    },
    rawBottom (newBottom) {
      const bounds = this.bounds
      const aspectFactor = this.aspectFactor
      const lockAspectRatio = this.lockAspectRatio
      const right = this.right
      const bottom = this.bottom

      if (bounds.minBottom !== null && newBottom < bounds.minBottom) {
        newBottom = bounds.minBottom
      } else if (bounds.maxBottom !== null && bounds.maxBottom < newBottom) {
        newBottom = bounds.maxBottom
      }

      if (lockAspectRatio && this.resizingOnY) {
        this.rawRight = right - (bottom - newBottom) * aspectFactor
      }

      this.bottom = newBottom
    },
    x () {
      if (this.resizing || this.dragging) {
        return
      }

      if (this.parent) {
        this.bounds = this.calcDragLimits()
      }

      const delta = this.x - this.left

      if (delta % this.grid[0] === 0) {
        this.rawLeft = this.x
        this.rawRight = this.right - delta
      }
    },
    y () {
      if (this.resizing || this.dragging) {
        return
      }

      if (this.parent) {
        this.bounds = this.calcDragLimits()
      }

      const delta = this.y - this.top

      if (delta % this.grid[1] === 0) {
        this.rawTop = this.y
        this.rawBottom = this.bottom - delta
      }
    },
    lockAspectRatio (val) {
      if (val) {
        this.aspectFactor = this.width / this.height
      } else {
        this.aspectFactor = undefined
      }
    },
    minWidth (val) {
      if (val > 0 && val <= this.width) {
        this.minW = val
      }
    },
    minHeight (val) {
      if (val > 0 && val <= this.height) {
        this.minH = val
      }
    },
    maxWidth (val) {
      this.maxW = val
    },
    maxHeight (val) {
      this.maxH = val
    },
    w () {
      if (this.resizing || this.dragging) {
        return
      }

      if (this.parent) {
        this.bounds = this.calcResizeLimits()
      }

      const delta = this.width - this.w

      if (delta % this.grid[0] === 0) {
        this.rawRight = this.right + delta
      }
    },
    h () {
      // -by Adang 调整文本组件宽度时更新高度
      // if (this.resizing || this.dragging) {
      //   return;
      // }

      if (this.parent) {
        this.bounds = this.calcResizeLimits()
      }

      const delta = this.height - this.h

      if (delta % this.grid[1] === 0) {
        this.rawBottom = this.bottom + delta
      }
    }
  },

  created () {
    // eslint-disable-next-line
    if (this.maxWidth && this.minWidth > this.maxWidth) console.warn('[Vdr warn]: Invalid prop: minWidth cannot be greater than maxWidth');
    // eslint-disable-next-line
    if (this.maxWidth && this.minHeight > this.maxHeight) console.warn('[Vdr warn]: Invalid prop: minHeight cannot be greater than maxHeight');

    this.resetBoundsAndMouseState()
  },
  mounted () {
    if (!this.enableNativeDrag) {
      this.$el.ondragstart = () => false
    }

    [this.parentWidth, this.parentHeight] = this.getParentSize()

    this.rawRight = this.parentWidth - this.rawWidth - this.rawLeft
    this.rawBottom = this.parentHeight - this.rawHeight - this.rawTop

    // +by Adang
    this.settingAttribute()

    addEvent(this.deselectElement ? document.querySelector(this.deselectElement) : document.documentElement, 'mousedown', this.deselect)
    addEvent(this.deselectElement ? document.querySelector(this.deselectElement) : document.documentElement, 'touchend touchcancel', this.deselect)

    addEvent(window, 'resize', this.checkParentSize)
  },
  beforeDestroy () {
    removeEvent(this.deselectElement ? document.querySelector(this.deselectElement) : document.documentElement, 'mousedown', this.deselect)
    removeEvent(document.documentElement, 'touchstart', this.handleUp)
    removeEvent(document.documentElement, 'mousemove', this.move)
    removeEvent(document.documentElement, 'touchmove', this.move)
    removeEvent(document.documentElement, 'mouseup', this.handleUp)
    removeEvent(this.deselectElement ? document.querySelector(this.deselectElement) : document.documentElement, 'touchend touchcancel', this.deselect)

    removeEvent(window, 'resize', this.checkParentSize)
  },

  methods: {
    resetBoundsAndMouseState () {
      this.mouseClickPosition = { mouseX: 0, mouseY: 0, x: 0, y: 0, w: 0, h: 0 }

      this.bounds = {
        minLeft: null,
        maxLeft: null,
        minRight: null,
        maxRight: null,
        minTop: null,
        maxTop: null,
        minBottom: null,
        maxBottom: null
      }
    },
    checkParentSize () {
      if (this.parent) {
        const [newParentWidth, newParentHeight] = this.getParentSize()

        const deltaX = this.parentWidth - newParentWidth
        const deltaY = this.parentHeight - newParentHeight

        this.rawRight -= deltaX
        this.rawBottom -= deltaY

        this.parentWidth = newParentWidth
        this.parentHeight = newParentHeight
      }
    },
    getParentSize () {
      if (this.parent) {
        const style = window.getComputedStyle(this.$el.parentNode, null)

        return [
          parseInt(style.getPropertyValue('width'), 10),
          parseInt(style.getPropertyValue('height'), 10)
        ]
      }

      return [null, null]
    },
    elementTouchDown (e) {
      eventsFor = events.touch

      this.elementDown(e)
    },
    elementDown (e) {
      const target = e.target || e.srcElement

      if (this.$el.contains(target)) {
        if (this.onDragStart && this.onDragStart(e) === false) {
          return
        }

        if (
          (this.dragHandle && !matchesSelectorToParentElements(target, this.dragHandle, this.$el)) ||
          (this.dragCancel && matchesSelectorToParentElements(target, this.dragCancel, this.$el))
        ) {
          return
        }

        if (!this.enabled) {
          this.enabled = true

          this.$emit('activated')
          this.$emit('update:active', true)
        }

        if (this.draggable) {
          this.dragging = true
        }

        this.mouseClickPosition.mouseX = e.touches ? e.touches[0].pageX : e.pageX
        this.mouseClickPosition.mouseY = e.touches ? e.touches[0].pageY : e.pageY

        this.mouseClickPosition.left = this.left
        this.mouseClickPosition.right = this.right
        this.mouseClickPosition.top = this.top
        this.mouseClickPosition.bottom = this.bottom

        if (this.parent) {
          this.bounds = this.calcDragLimits()
        }

        addEvent(document.documentElement, eventsFor.move, this.move)
        addEvent(document.documentElement, eventsFor.stop, this.handleUp)
      }
    },
    calcDragLimits () {
      return {
        minLeft: (this.parentWidth + this.left) % this.grid[0],
        maxLeft: Math.floor((this.parentWidth - this.width - this.left) / this.grid[0]) * this.grid[0] + this.left,
        minRight: (this.parentWidth + this.right) % this.grid[0],
        maxRight: Math.floor((this.parentWidth - this.width - this.right) / this.grid[0]) * this.grid[0] + this.right,
        minTop: (this.parentHeight + this.top) % this.grid[1],
        maxTop: Math.floor((this.parentHeight - this.height - this.top) / this.grid[1]) * this.grid[1] + this.top,
        minBottom: (this.parentHeight + this.bottom) % this.grid[1],
        maxBottom: Math.floor((this.parentHeight - this.height - this.bottom) / this.grid[1]) * this.grid[1] + this.bottom
      }
    },
    // 设置选中
    setActive () {
      this.enabled = true
      this.$emit('update:active', true)
    },
    // 停用
    deselect (e) {
      if (e) {
        // 主动行为
        const target = e.target || e.srcElement
        const regex = new RegExp(this.className + '-([trmbl]{2})', '')

        if (!this.$el.contains(target) && !regex.test(target.className)) {
          if (this.enabled && !this.preventDeactivation) {
            this.enabled = false

            this.$emit('deactivated')
            this.$emit('update:active', false)
          }
        }
      } else {
        // 立法调用
        this.enabled = false
        this.$emit('deactivated')
        this.$emit('update:active', false)
      }

      this.resetBoundsAndMouseState()
      removeEvent(document.documentElement, eventsFor.move, this.handleMove)
    },
    // +by Adang
    handleRotateDown (event) {
      addEvent(document.documentElement, 'mousemove', this.handleRotateMove)
      addEvent(document.documentElement, 'mouseup', () => {
        removeEvent(document.documentElement, 'mousemove', this.handleRotateMove)
      })
      this.handleRotateStart(event)
    },

    // 旋转开始 +by Adang
    handleRotateStart (event) {
      const { clientX, clientY } = event.touches ? event.touches[0] : event
      const t = this.$refs['wrapper'].getBoundingClientRect()
      const cx = t.left + t.width / 2
      const cy = t.top + t.height / 2
      const startAngle = (180 / Math.PI) * Math.atan2(clientY - cy, clientX - cx)
      const rotation = this.rawRotate
      this.rotateOpt = { cx, cy, startAngle, rotation }
    },

    // 旋转 +by Adang
    handleRotateMove (event) {
      const { cx, cy, startAngle, rotation } = this.rotateOpt
      const { clientX, clientY } = event.touches ? event.touches[0] : event
      const x = clientX - cx
      const y = clientY - cy
      const angle = (180 / Math.PI) * Math.atan2(y, x)
      const currentAngle = angle - startAngle
      let r = rotation + currentAngle
      r = r % 360
      r = r < 0 ? r + 360 : r
      this.rawRotate = Math.floor(r)

      this.$emit('on-rotate', this.rotate)
    },
    // 控制柄触摸按下
    handleTouchDown (handle, e) {
      eventsFor = events.touch

      this.handleDown(handle, e)
    },

    handleDown (handle, e) {
      if (this.onResizeStart && this.onResizeStart(handle, e) === false) {
        return
      }

      if (e.stopPropagation) e.stopPropagation()

      // Here we avoid a dangerous recursion by faking
      // corner handles as middle handles
      this.currentHandle = handle
      if (this.lockAspectRatio && !handle.includes('m')) {
        this.handle = 'm' + handle.substring(1)
      } else {
        this.handle = handle
      }

      this.resizing = true

      this.mouseClickPosition.mouseX = e.touches ? e.touches[0].pageX : e.pageX
      this.mouseClickPosition.mouseY = e.touches ? e.touches[0].pageY : e.pageY

      this.mouseClickPosition.left = this.left
      this.mouseClickPosition.right = this.right
      this.mouseClickPosition.top = this.top
      this.mouseClickPosition.bottom = this.bottom

      this.bounds = this.calcResizeLimits()

      addEvent(document.documentElement, eventsFor.move, this.handleMove)
      addEvent(document.documentElement, eventsFor.stop, this.handleUp)
    },
    calcResizeLimits () {
      let minW = this.minW
      let minH = this.minH
      let maxW = this.maxW
      let maxH = this.maxH

      const aspectFactor = this.aspectFactor
      const [gridX, gridY] = this.grid
      const width = this.width
      const height = this.height
      const left = this.left
      const top = this.top
      const right = this.right
      const bottom = this.bottom

      if (this.lockAspectRatio) {
        if (minW / minH > aspectFactor) {
          minH = minW / aspectFactor
        } else {
          minW = aspectFactor * minH
        }

        if (maxW && maxH) {
          maxW = Math.min(maxW, aspectFactor * maxH)
          maxH = Math.min(maxH, maxW / aspectFactor)
        } else if (maxW) {
          maxH = maxW / aspectFactor
        } else if (maxH) {
          maxW = aspectFactor * maxH
        }
      }

      maxW = maxW - (maxW % gridX)
      maxH = maxH - (maxH % gridY)

      const limits = {
        minLeft: null,
        maxLeft: null,
        minTop: null,
        maxTop: null,
        minRight: null,
        maxRight: null,
        minBottom: null,
        maxBottom: null
      }

      if (this.parent) {
        limits.minLeft = (this.parentWidth + left) % gridX
        limits.maxLeft = left + Math.floor((width - minW) / gridX) * gridX
        limits.minTop = (this.parentHeight + top) % gridY
        limits.maxTop = top + Math.floor((height - minH) / gridY) * gridY
        limits.minRight = (this.parentWidth + right) % gridX
        limits.maxRight = right + Math.floor((width - minW) / gridX) * gridX
        limits.minBottom = (this.parentHeight + bottom) % gridY
        limits.maxBottom = bottom + Math.floor((height - minH) / gridY) * gridY

        if (maxW) {
          limits.minLeft = Math.max(limits.minLeft, this.parentWidth - right - maxW)
          limits.minRight = Math.max(limits.minRight, this.parentWidth - left - maxW)
        }

        if (maxH) {
          limits.minTop = Math.max(limits.minTop, this.parentHeight - bottom - maxH)
          limits.minBottom = Math.max(limits.minBottom, this.parentHeight - top - maxH)
        }

        if (this.lockAspectRatio) {
          limits.minLeft = Math.max(limits.minLeft, left - top * aspectFactor)
          limits.minTop = Math.max(limits.minTop, top - left / aspectFactor)
          limits.minRight = Math.max(limits.minRight, right - bottom * aspectFactor)
          limits.minBottom = Math.max(limits.minBottom, bottom - right / aspectFactor)
        }
      } else {
        limits.minLeft = null
        limits.maxLeft = left + Math.floor((width - minW) / gridX) * gridX
        limits.minTop = null
        limits.maxTop = top + Math.floor((height - minH) / gridY) * gridY
        limits.minRight = null
        limits.maxRight = right + Math.floor((width - minW) / gridX) * gridX
        limits.minBottom = null
        limits.maxBottom = bottom + Math.floor((height - minH) / gridY) * gridY

        if (maxW) {
          limits.minLeft = -(right + maxW)
          limits.minRight = -(left + maxW)
        }

        if (maxH) {
          limits.minTop = -(bottom + maxH)
          limits.minBottom = -(top + maxH)
        }

        if (this.lockAspectRatio && (maxW && maxH)) {
          limits.minLeft = Math.min(limits.minLeft, -(right + maxW))
          limits.minTop = Math.min(limits.minTop, -(maxH + bottom))
          limits.minRight = Math.min(limits.minRight, -left - maxW)
          limits.minBottom = Math.min(limits.minBottom, -top - maxH)
        }
      }

      return limits
    },
    move (e) {
      if (this.resizing) {
        this.handleMove(e)
      } else if (this.dragging) {
        this.elementMove(e)
      }
    },
    elementMove (e) {
      const axis = this.axis
      // const grid = this.grid;
      const mouseClickPosition = this.mouseClickPosition

      const tmpDeltaX = axis && axis !== 'y' ? mouseClickPosition.mouseX - (e.touches ? e.touches[0].pageX : e.pageX) : 0
      const tmpDeltaY = axis && axis !== 'x' ? mouseClickPosition.mouseY - (e.touches ? e.touches[0].pageY : e.pageY) : 0

      const [deltaX, deltaY] = this.snapToGrid(this.grid, tmpDeltaX, tmpDeltaY)

      this.rawTop = mouseClickPosition.top - deltaY
      this.rawBottom = mouseClickPosition.bottom + deltaY
      this.rawLeft = mouseClickPosition.left - deltaX
      this.rawRight = mouseClickPosition.right + deltaX

      // +by Adang 对齐线
      this.snapCheck()

      this.$emit('dragging', this.left, this.top)
    },
    handleMove (e) {
      const handle = this.handle
      const mouseClickPosition = this.mouseClickPosition

      const tmpDeltaX = mouseClickPosition.mouseX - (e.touches ? e.touches[0].pageX : e.pageX)
      const tmpDeltaY = mouseClickPosition.mouseY - (e.touches ? e.touches[0].pageY : e.pageY)

      const [deltaX, deltaY] = this.snapToGrid(this.grid, tmpDeltaX, tmpDeltaY)

      if (handle.includes('b')) {
        this.rawBottom = mouseClickPosition.bottom + deltaY
      } else if (handle.includes('t')) {
        this.rawTop = mouseClickPosition.top - deltaY
      }

      if (handle.includes('r')) {
        this.rawRight = mouseClickPosition.right + deltaX
      } else if (handle.includes('l')) {
        this.rawLeft = mouseClickPosition.left - deltaX
      }

      this.$emit('resizing', this.left, this.top, this.width, this.height)
    },
    // 从控制柄松开
    handleUp () {
      this.handle = null

      this.resetBoundsAndMouseState()

      this.rawTop = this.top
      this.rawBottom = this.bottom
      this.rawLeft = this.left
      this.rawRight = this.right

      // +by Adang 初始化辅助线数据
      const temArr = new Array(3).fill({ display: false, position: '', origin: '', lineLength: '' })
      const refLine = { vLine: [], hLine: [] }
      for (const i in refLine) { refLine[i] = JSON.parse(JSON.stringify(temArr)) }

      if (this.resizing) {
        this.resizing = false
        this.$emit('on-line-params', refLine)
        this.$emit('resizestop', this.left, this.top, this.width, this.height)
      }
      if (this.dragging) {
        this.dragging = false
        this.$emit('on-line-params', refLine)
        this.$emit('dragstop', this.left, this.top)
      }

      removeEvent(document.documentElement, eventsFor.move, this.handleMove)
    },
    snapToGrid (grid, pendingX, pendingY) {
      const x = Math.round((pendingX / this.scale) / grid[0]) * grid[0]
      const y = Math.round((pendingY / this.scale) / grid[1]) * grid[1]

      return [x, y]
    },
    // 新增方法 ↓↓↓
    // 设置属性
    settingAttribute () {
      // 设置对齐元素
      this.snap ? this.$el.setAttribute('data-is-snap', 'true') : this.$el.setAttribute('data-is-snap', 'false')
    },
    // 检测对齐元素
    async snapCheck () {
      let width = this.width
      let height = this.height
      if (this.snap) {
        let activeLeft = this.rawLeft
        let activeRight = this.rawLeft + width
        let activeTop = this.rawTop
        let activeBottom = this.rawTop + height

        // 初始化辅助线数据
        const temArr = new Array(3).fill({ display: false, position: '', origin: '', lineLength: '' })
        const refLine = { vLine: [], hLine: [] }
        for (const i in refLine) { refLine[i] = JSON.parse(JSON.stringify(temArr)) }

        // 获取当前父节点下所有子节点
        const nodes = this.$el.parentNode.childNodes

        const tem = {
          value: { x: [[], [], []], y: [[], [], []] },
          display: [],
          position: []
        }
        const { groupWidth, groupHeight, groupLeft, groupTop, bln } = await this.getActiveAll(nodes)
        if (!bln) {
          width = groupWidth
          height = groupHeight
          activeLeft = groupLeft
          activeRight = groupLeft + groupWidth
          activeTop = groupTop
          activeBottom = groupTop + groupHeight
        }
        for (const item of nodes) {
          if (item.className !== undefined && !item.className.includes(this.classNameActive) && item.getAttribute('data-is-snap') !== null && item.getAttribute('data-is-snap') !== 'false') {
            const w = item.offsetWidth
            const h = item.offsetHeight
            const l = item.offsetLeft // 对齐目标的left
            const r = l + w // 对齐目标right
            const t = item.offsetTop// 对齐目标的top
            const b = t + h // 对齐目标的bottom

            const hc = Math.abs((activeTop + height / 2) - (t + h / 2)) <= this.snapTolerance // 水平中线
            const vc = Math.abs((activeLeft + width / 2) - (l + w / 2)) <= this.snapTolerance // 垂直中线

            const ts = Math.abs(t - activeBottom) <= this.snapTolerance // 从上到下
            const TS = Math.abs(b - activeBottom) <= this.snapTolerance // 从上到下
            const bs = Math.abs(t - activeTop) <= this.snapTolerance // 从下到上
            const BS = Math.abs(b - activeTop) <= this.snapTolerance // 从下到上

            const ls = Math.abs(l - activeRight) <= this.snapTolerance // 外左
            const LS = Math.abs(r - activeRight) <= this.snapTolerance // 外左
            const rs = Math.abs(l - activeLeft) <= this.snapTolerance // 外右
            const RS = Math.abs(r - activeLeft) <= this.snapTolerance // 外右

            tem['display'] = [ts, TS, bs, BS, hc, hc, ls, LS, rs, RS, vc, vc]
            tem['position'] = [t, b, t, b, t + h / 2, t + h / 2, l, r, l, r, l + w / 2, l + w / 2]

            if (ts) {
              if (bln) {
                this.rawTop = t - height
                this.rawBottom = this.parentHeight - this.rawTop - height
              }
              tem.value.y[0].push(l, r, activeLeft, activeRight)
            }
            if (bs) {
              if (bln) {
                this.rawTop = t
                this.rawBottom = this.parentHeight - this.rawTop - height
              }
              tem.value.y[0].push(l, r, activeLeft, activeRight)
            }
            if (TS) {
              if (bln) {
                this.rawTop = b - height
                this.rawBottom = this.parentHeight - this.rawTop - height
              }
              tem.value.y[1].push(l, r, activeLeft, activeRight)
            }
            if (BS) {
              if (bln) {
                this.rawTop = b
                this.rawBottom = this.parentHeight - this.rawTop - height
              }
              tem.value.y[1].push(l, r, activeLeft, activeRight)
            }

            if (ls) {
              if (bln) {
                this.rawLeft = l - width
                this.rawRight = this.parentWidth - this.rawLeft - width
              }
              tem.value.x[0].push(t, b, activeTop, activeBottom)
            }
            if (rs) {
              if (bln) {
                this.rawLeft = l
                this.rawRight = this.parentWidth - this.rawLeft - width
              }
              tem.value.x[0].push(t, b, activeTop, activeBottom)
            }
            if (LS) {
              if (bln) {
                this.rawLeft = r - width
                this.rawRight = this.parentWidth - this.rawLeft - width
              }
              tem.value.x[1].push(t, b, activeTop, activeBottom)
            }
            if (RS) {
              if (bln) {
                this.rawLeft = r
                this.rawRight = this.parentWidth - this.rawLeft - width
              }
              tem.value.x[1].push(t, b, activeTop, activeBottom)
            }

            if (hc) {
              if (bln) {
                this.rawTop = t + h / 2 - height / 2
                this.rawBottom = this.parentHeight - this.rawTop - height
              }
              tem.value.y[2].push(l, r, activeLeft, activeRight)
            }
            if (vc) {
              if (bln) {
                this.rawLeft = l + w / 2 - width / 2
                this.rawRight = this.parentWidth - this.rawLeft - width
              }
              tem.value.x[2].push(t, b, activeTop, activeBottom)
            }
            // 辅助线坐标与是否显示(display)对应的数组,易于循环遍历
            const arrTem = [0, 1, 0, 1, 2, 2, 0, 1, 0, 1, 2, 2]
            for (let i = 0; i <= arrTem.length; i++) {
              // 前6为Y辅助线,后6为X辅助线
              const xory = i < 6 ? 'y' : 'x'
              const horv = i < 6 ? 'hLine' : 'vLine'
              if (tem.display[i]) {
                const { origin, length } = this.calcLineValues(tem.value[xory][arrTem[i]])
                refLine[horv][arrTem[i]].display = tem.display[i]
                refLine[horv][arrTem[i]].position = tem.position[i] + 'px'
                refLine[horv][arrTem[i]].origin = origin
                refLine[horv][arrTem[i]].lineLength = length
              }
            }
          }
        }
        this.$emit('on-line-params', refLine)
      }
    },
    calcLineValues (arr) {
      const length = Math.max(...arr) - Math.min(...arr) + 'px'
      const origin = Math.min(...arr) + 'px'
      return { length, origin }
    },
    async getActiveAll (nodes) {
      const activeAll = []
      const XArray = []
      const YArray = []
      let groupWidth = 0
      let groupHeight = 0
      let groupLeft = 0
      let groupTop = 0
      for (const item of nodes) {
        if (item.className !== undefined && item.className.includes(this.classNameActive)) {
          activeAll.push(item)
        }
      }
      const AllLength = activeAll.length
      if (AllLength > 1) {
        for (const i of activeAll) {
          const l = i.offsetLeft
          const r = l + i.offsetWidth
          const t = i.offsetTop
          const b = t + i.offsetHeight
          XArray.push(t, b)
          YArray.push(l, r)
        }
        groupWidth = Math.max(...YArray) - Math.min(...YArray)
        groupHeight = Math.max(...XArray) - Math.min(...XArray)
        groupLeft = Math.min(...YArray)
        groupTop = Math.min(...XArray)
      }
      const bln = AllLength === 1
      return { groupWidth, groupHeight, groupLeft, groupTop, bln }
    }
  }
}
</script>
<style lang="less">
  .x-draggable-resizable {
    touch-action: none;
    position: absolute;
    box-sizing: border-box;
    border: 1px dashed black;
    .handle {
      box-sizing: border-box;
      position: absolute;
      width: 10px;
      height: 10px;
      background: #EEE;
      border: 1px solid #333;
    }
    .handle-tl {
      top: -10px;
      left: -10px;
      cursor: nw-resize;
    }
    .handle-tm {
      top: -10px;
      left: 50%;
      margin-left: -5px;
      cursor: n-resize;
    }
    .handle-tr {
      top: -10px;
      right: -10px;
      cursor: ne-resize;
    }
    .handle-ml {
      top: 50%;
      margin-top: -5px;
      left: -10px;
      cursor: w-resize;
    }
    .handle-mr {
      top: 50%;
      margin-top: -5px;
      right: -10px;
      cursor: e-resize;
    }
    .handle-bl {
      bottom: -10px;
      left: -10px;
      cursor: sw-resize;
    }
    .handle-bm {
      bottom: -10px;
      left: 50%;
      margin-left: -5px;
      cursor: s-resize;
    }
    .handle-br {
      bottom: -10px;
      right: -10px;
      cursor: se-resize;
    }
    // +by Adang
    // 旋转handle
    .handle-rotate {
      bottom: 0;
      left: 50%;
      margin-left: -1px;
      margin-bottom: -40px;
      width: 19px;
      height: 19px;
      cursor: grab;
      border: 0;
      transform: translate(-50%, 0);
      background: #fff;
      border-radius: 50%;
      box-shadow: 0 0 2px #bbb;
      i{ z-index: 2; position: relative; padding: 3px; background: #fff; border-radius: 50%;}
      &:after{ position: absolute; content: ''; width: 1px; height: 20px; left: 50%; top: -18px; border-left: 1px dashed #6ccfff;}
    }
    @media only screen and (max-width: 768px) {
      [class*="handle-"]:before {
        content: '';
        left: -10px;
        right: -10px;
        bottom: -10px;
        top: -10px;
        position: absolute;
      }
    }

  }
</style>
