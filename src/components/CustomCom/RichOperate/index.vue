<template>
  <div class="rich-operate">
    <slot></slot>
  </div>
</template>

<script>
export default {
  name: 'RichOperate',
  // 分发操作数据
  provide() {
    return {
      // 添加节点数据
      dhtSetList: ({
        bindData = null, // 绑定数据
        self = null, // 本身
        select = false, // 是否被选中
      }) => {
        let alike = false
        // 判断是否已经存在数据列表内部
        this.list.forEach(item => {
          if (item.self === self) {
            item.bindData = bindData
            alike = true
          }
        })
        if (!alike) {
          this.list.push({
            bindData,
            self,
            select,
          })
        }
        // console.log(this.list, this.list.length)
      },
    }
  },
  props: {
    // 结构为上下左右，传入0为关闭，传入对应键位则为设置键位响应
    keyPress: {
      type: Array,
      default() {
        return [38, 40, 37, 39] // 上下左右
      },
    },
    // 初始index
    index: {
      type: Number,
      default: 0,
    },
    type: {
      type: String,
      default: 'direction', //direction：键盘方向键， letter:字母wasd模式,custom: 自定义模式
    },
    background: {
      type: String,
      default: 'rgba(5, 173, 255, 0.5)',
    },
    switch: Boolean, // 开关键盘事件
  },
  watch: {
    index: {
      immediate: true,
      handler(index) {
        let id = setInterval(() => {
          const li = this.list[index]
          if (li) {
            this.sendEmit({ item: li, index })
            clearInterval(id)
          }
        }, 200)
      },
    },
    switch: {
      immediate: true,
      handler(value) {
        if (value) {
          window.addEventListener('keydown', this.onKeyDown, false)
        }
        if (!value) {
          window.removeEventListener('keydown', this.onKeyDown, false)
        }
      },
    },
  },
  data() {
    return {
      list: [], // 操作列表数据
      currentIndex: 0, // 当前被选中的数据
    }
  },
  created() {
    // 监听按键事件
    this.$once('hook:beforeDestroy', () => {
      window.removeEventListener('keydown', this.onKeyDown, false)
    })
  },
  mounted() {},
  methods: {
    // 键盘监听事件
    onKeyDown(e) {
      const code = e.keyCode
      const [top, bottom, left, right] = this.keyPress
      const type = {
        [top]: 'top', // 上
        [bottom]: 'bottom', // 下
        [left]: 'left', // 左
        [right]: 'right', // 右
      }
      this.move(type[code])
    },
    // 移动点
    move(type) {
      if (type === 'top') this.topOrbottomMove(type)
      if (type === 'bottom') this.topOrbottomMove(type)
      if (type === 'left') this.leftOrRightMove(type)
      if (type === 'right') this.leftOrRightMove(type)
    },
    // 上一次元素改为无选中状态
    lastTimeLi() {
      const lastTimeIndex = this.currentIndex // 上一次的
      this.list[lastTimeIndex].self.$el.style.background = ''
      this.list[lastTimeIndex].select = false // 上一次的变为未选中
      return lastTimeIndex
    },
    // 左右移动
    leftOrRightMove(type) {
      let nextIndex = this.lastTimeLi() // 上一次的
      const list = this.list
      const maxIndex = list.length // 最大
      if (type === 'left') {
        nextIndex--
        nextIndex = nextIndex <= 0 ? 0 : nextIndex
      }
      if (type === 'right') {
        nextIndex++
        nextIndex = nextIndex >= maxIndex ? maxIndex - 1 : nextIndex
      }
      // 最终发送确认值
      this.sendEmit({ item: list[nextIndex], index: nextIndex })
    },
    // 上下移动
    topOrbottomMove(type) {
      let currentIndex = this.lastTimeLi() // 上一次的index,也是当前的index
      const list = this.list
      // 当前
      const currentLi = list[currentIndex]
      const currentLiDom = currentLi.self.$el.getBoundingClientRect()
      const currentY = currentLiDom.y
      const currentX = currentLiDom.x

      const relativeList = []
      list.forEach((item, index) => {
        const { y, x } = item.self.$el.getBoundingClientRect()
        relativeList.push({
          li: item,
          y: currentY - y,
          x: Math.abs(currentX - x),
          index,
        })
      })
      // 过滤通等级元素
      const eliminate = relativeList.filter(item => item.y !== 0)
      if (type === 'top') {
        // 往上过滤比自己低的
        const topEliminate = eliminate.filter(item => item.y > 0)
        if (topEliminate.length === 0) {
          // 最终发送确认值
          this.sendEmit({ item: list[currentIndex], index: currentIndex })
        } else {
          topEliminate.sort((a, b) => a.y - b.y)
          const xArr = topEliminate.filter(item => item.y === topEliminate[0].y)
          if (xArr.length > 1) {
            xArr.sort((a, b) => a.x - b.x)
            this.sendEmit({ item: xArr[0], index: xArr[0].index })
          } else {
            this.sendEmit({ item: xArr[0], index: xArr[0].index })
          }
        }
      }
      if (type === 'bottom') {
        // 往下过滤比自己高的
        const bottomEliminate = eliminate.filter(item => item.y < 0)
        if (bottomEliminate.length === 0) {
          // 最终发送确认值
          this.sendEmit({ item: list[currentIndex], index: currentIndex })
        } else {
          bottomEliminate.sort((a, b) => b.y - a.y)
          const xArr = bottomEliminate.filter(item => item.y === bottomEliminate[0].y)
          if (xArr.length > 1) {
            xArr.sort((a, b) => a.x - b.x)
            this.sendEmit({ item: xArr[0], index: xArr[0].index })
          } else {
            this.sendEmit({ item: xArr[0], index: xArr[0].index })
          }
        }
      }
    },
    // 统一返回给外界当前数据
    sendEmit({ item, index }) {
      // 操作当前选中的dom或者实例
      this.list[index].select = true
      this.currentIndex = index
      const background = this.background
      if (background) {
        const dom = this.list[index].self.$el
        dom.style.background = this.background
      }

      this.$emit('change', { item, index })
    },
  },
}
</script>

<style lang="scss">
.rich-operate {
  background: rgba(5, 173, 255, 0.1);
}
</style>
