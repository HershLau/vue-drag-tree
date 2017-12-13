<template>
    <div>
        <div :id='model.id' @click="toggle" draggable='true' @dragstart='dragStart' @dragover='dragOver' @drop='drop' @dragend='dragEnd' class='treeNodeText' :style='styleObj'>
      <span v-show="model.children&&model.children.length"
            :class="[isClicked ? 'nodeClicked' : '','vue-drag-node-icon']"></span>
            {{model.name}}
        </div>
        <div class='treeMargin' v-show="open" v-if="isFolder">
            <item v-for="model in model.children" :model="model" :key='model.id' :cur-node-clicked="curNodeClicked"
                  :assign-data="assignData" :delete-node="deleteNode" :current-highlight='currentHighlight'
                  :default-text='defaultText' :hover-color='hoverColor' :highlight-color='highlightColor' :isOpen="open">
            </item>
        </div>
    </div>
</template>

<script type="text/ecmascript-6">
  let fromData = ''
  let toData = ''
  let nodeClicked = undefined

  export default {
    name: 'Item',
    data() {
      return {
        open: this.isOpen,
        isClicked: this.isOpen,
        styleObj: {
          background: 'white'
        }
      }
    },
    computed: {
      isFolder() {
        return this.model.children &&
            this.model.children.length
      },
    },
    props: {
      model: Object,
      'isOpen': Boolean,
      'default-text': String, // 填加节点时显示的默认文本．
      'current-highlight': Boolean, // 当前节点高亮
      'hover-color': String,
      'highlight-color': String,
      'cur-node-clicked': Function,
      'assign-data': Function,
      'delete-node': Function
    },
    methods: {
      refresh() {
        let nodeStack = [this.findRoot()]
        while (nodeStack.length != 0) {
          let item = nodeStack.shift()
          item.styleObj.background = 'white'
          item.open = this.isOpen
          item.isClicked = this.isOpen
          if (item.$children && item.$children.length > 0) {
            nodeStack = nodeStack.concat(item.$children)
          }
        }
      },
      dragStart(e) {
        fromData = this.model
        e.dataTransfer.effectAllowed = "move"
        e.dataTransfer.setData("nottext", e.target.innerHTML)
        return true
      },
      dragOver(e) {
        e.preventDefault()
        return true
      },
      drop(e) {
        toData = this.model
        let rootTree = this.findRoot()
        rootTree.exchangeData(this, fromData, toData)
      },
      dragEnd(e) {
        fromData = undefined
        toData = undefined
      },
      toggle() {
        if (this.isFolder) {
          this.open = !this.open
        }
        // 调用vue-drag-tree的父组件中的方法,以传递出当前被点击的节点的id值
        let rootTree = this.findRoot()
        //　API: 对外开放的当前被点击节点的信息
        this.curNodeClicked(this.model, this)

        // 纪录节点被点击的状态
        this.isClicked = !this.isClicked

        // 用户需要节点高亮？　-->　this.currentHighlight ? 高亮 : 不高亮
        if (this.currentHighlight) {
          // 第一次点击当前节点．当前节点高亮，遍历重置其他节点的样式
          if (nodeClicked != this.model.id) {
            // 遍历重置所有树组件的高亮样式
            let nodeStack = [rootTree]
            while (nodeStack.length != 0) {
              let item = nodeStack.shift()
              item.styleObj.background = 'white'
              if (item.$children && item.$children.length > 0) {
                nodeStack = nodeStack.concat(item.$children)
              }
            }
            // 然后把当前节点的样式设置为高亮
            this.styleObj.background = this.highlightColor ? this.highlightColor : '#E5E9F2'

            // 设置为当前节点
            nodeClicked = this.model.id
          }
        }
      },
      findRoot() {
        // 返回Tree的根,即递归Tree时的最顶层那个vue-drag-tree组件
        let ok = false
        let that = this
        while (!ok) {
          if (!/Item/.test(that.$parent.$vnode.tag) || that.$parent.data) {
            ok = true
            break
          }
          that = that.$parent
        }
        return that
      },
      exchangeData(rootCom, from, to) {
        from = Object.assign({},from)
        to = Object.assign({},to)
        // copy一个,最后再赋值给state.treeData.这样能保证值的变化都会触发视图刷新(因为JS判断引用类型是否相同是根据内存地址.)
        let treeData = this.model
        let nodeStack = [treeData]
        let fromStack = [from]
        let toStack = [to]
        let status = 0

        let traverse = function (nodeStack, id) {
          let flag = false
          while (nodeStack.length) {
            let item = nodeStack.shift()
            if (item.id === id) {
              flag = true
            }
            if (item.children && item.children.length > 0) {
              nodeStack = nodeStack.concat(item.children)
            }
          }
          return flag
        }
        if (traverse(fromStack, to.id) || traverse(toStack, from.id)) {
          console.error('两节点存在包含关系，无法移动')
          return
        }

        // 如果from或者to节点存在父子/祖辈关系，会报id of undefined的错。这种情况不考虑拖拽功能，所以catch住，返回/return就行
        try {
          // 广度优先遍历,找到涉及数据交换的两个对象.然后交换数据.
          while (!(status === 2)) {
            let item = nodeStack.shift()
            if (item.id == from.id) {
              item.id = to.id
              item.name = to.name
              if (to.children && to.children.length > 0) {
                item['children'] = to.children
              } else {
                item.children = []
              }
              status++
              //找到后,跳出当前循环.
              continue;
            }
            if (item.id == to.id) {
              item.id = from.id
              item.name = from.name
              if (from.children && from.children.length > 0) {
                item['children'] = from.children
              } else {
                item.children = []
              }
              status++
              //找到后,跳出当前循环.
              continue;
            }
            if (item.children && item.children.length > 0) {
              nodeStack = nodeStack.concat(item.children)
            }
          }
        } catch (e) {
          return
        }
        //API: 对外开放交换后的数据的赋值操作
        rootCom.assignData(treeData, from, to)
      },
    }
  }
</script>

<style lang="stylus" rel="stylesheet/stylus">
    .item {
        cursor: pointer;
    }

    .bold {
        font-weight: bold;
    }

    .treeNodeText {
        cursor: pointer;
        margin: 2px;
        padding: 0.2rem 0.5rem;
        width: fit-content;
        background: #F9FAFC;
        font-size: 18px;
        color: #324057;
    }

    .treeMargin {
        margin-left: 2rem;
    }

    .changeTree {
        width: 1rem;
        color: #324057;
    }

    .vue-drag-node-icon {
        display: inline-block;
        width: 0;
        height: 0;
        padding-right: 3px;
        border-left: 6px solid black;
        border-top: 6px solid transparent;
        border-bottom: 6px solid transparent;
        border-right: 0 solid yellow;
        transition: transform .3s ease-in-out;
    }

    .nodeClicked {
        transform: rotate(90deg);
    }
</style>
