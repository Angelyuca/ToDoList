<template>
  <div>
    <note v-if="data2.activeEditor" :data="text" class="mb-2" @save="$emit('save', {list: list, text: $event})"/>
    <div class="card-tabs mb-2 pb-1 px-3 mx-n3">
      <div
          class="item mr-4"
          :class="{ active: activeTab === item.id }"
          v-for="(item, i) in tabs"
          :key="i"
          @click="activeTab = item.id"
      >
        <div
            class="tabs-title"
            v-html="item.title"
        >
        </div>
      </div>
    </div>

    <div
        class="todo-list mx-n3 wr-sc"
        v-show="list.length"
        :style="{'max-height': data2.activeEditor? 'calc(100% - 130px)': 'calc(100% - 100px)'}"
    >
      <liquor-tree
          ref="tree"
          class="custom"
          :data="treeData"
          :options="treeOptions"
          @node:dragging:finish="dragging"
          @node:unchecked="updateItem"
          @node:checked="updateItem"
          @node:expanded="updateItem"
          @node:collapsed="updateItem"
      >
        <div slot-scope="{ node }">
          <todo-item :item="node" @remove="remove($event)" @addSub="addSub"
                     @edit="updateItem($event, true)"></todo-item>
        </div>
      </liquor-tree>
    </div>
    <div v-show="!list.length" class="not-list">
      Нет задач
    </div>
    <div class="d-flex align-center mt-1">
      <div v-if="list.length" class="pseudo-checkbox"></div>
   
      <miniEditor
          style="width: 100%"
          :pasteHelper="pasteHelper"
          :placeholder="$t('fields.add_new_task')"
          @enter="enter"
      />
    </div>

  </div>
</template>

<script>
import LiquorTree from 'liquor-tree'
import todoItem from "./todoItem";
import note from "./note"
import arrayToTree from "array-to-tree";
import cloneDeep from "clone-deep";
import maxBy from "lodash.maxby";
import miniEditor from "@/components/redactors/miniEditor/miniEditor";

export default {
  name: 'App',
  props: {
    data: {
      default: [],
    },
    data2: {
      default: {},
    }
  },
  components: {
    note,
    LiquorTree,
    todoItem,
    miniEditor,
  },

  data() {
    return {
      list: [],
      text: '',
      activeTab: 0,
      input: "",
      edit: false,
      treeData: [],
      treeOptions: {
        propertyNames: {
          text: 'text',
          children: 'children',
        },
        dnd: true,
        checkbox: true,
        keyboardNavigation: false,
        autoCheckChildren: false,
        parentSelect: true
      }
    }
  },
  created() {
    if (!Array.isArray(this.data)) {
      let currentData = cloneDeep(this.data)
      this.list = currentData.list
      this.text = currentData.text
      this.$emit("totalInfo", this.list.length);
      this.treeData = arrayToTree(this.list)
    }
  },
  computed: {
    tabs() {
      return [
        {
          title:  this.$t('fields.all_tasks') + " (" + this.list.length + ")",
          id: 0,
        },
        {
          title:  this.$t('fields.unfulfilled') + " ("+ this.list.filter((o) => o.state.checked === false).length + ")",
          id: 1,
        },
        {
          title: this.$t('fields.completed') + " ("+ this.list.filter((o) => o.state.checked === true).length + ")",
          id: 2,
        },
      ]
    },
  },
  watch: {
    activeTab() {
      this.updateData()
    },
    data() {
      this.$emit("totalInfo", this.list.length);
    }
  },
  methods: {
    enter(data, parent_id = 0) {
      let newItem = {}
   
      for (let value of data) {
        if (value.length) {
          if (value.match(/<(\/?[^>]+)>/)) {
            let newDiv = document.createElement("div");
            newDiv.innerHTML = value;
            let objTags = newDiv.getElementsByTagName("*")
            if(objTags.length>2) {
              console.log(objTags)
              for (let i = 1; i < objTags.length; ++i) {
                if(objTags[i].innerText.length) this.addList(objTags[i].innerText, parent_id)
              }
            } else this.addList(objTags[0].innerText, parent_id)
          } else this.addList(value, parent_id)

        }
      }
      this.$emit("save", {list: this.list, text: this.text})
      this.updateData()
    },

    addList(value, parent_id){
      let maxId = maxBy(this.list, function (o) {
        return o.id;
      })
      let newItem = {
        id: maxId ? maxId.id + 1 : 1,
        text: value,
        parent_id: parent_id,
        state: {checked: false, expanded: true}
      }
      this.list.push(newItem);
      if (parent_id > 0) {
        this.list = this.list.map((i) => {
          if (i.id === parent_id) {
            i.state.expanded = true
            return i
          } else return i
        })
      }
    },
    async pasteHelper(data) {
      const resp = await this.$store.dispatch("floppyUser/uploadClipboard", {
        data,
      });

      if (resp.message) {
        this.respError(resp);
        return false;
      }
      return resp;
    },
    addSub(e) {
      this.enter(e.text, e.parent_id)
    },
    updateData() {
      if (!this.list) {
        this.$refs.tree.tree.setModel([])
        return
      }
      if (!this.activeTab) {
        this.$refs.tree.tree.setModel(arrayToTree(this.list))
        return
      }
      let done = false;
      if (this.activeTab === 2) {
        done = true;
      }
      this.$refs.tree.tree.setModel(arrayToTree(this.list.filter((o) => o.state.checked === done)));
    },
    updateItem(e, update = false) {
      let item = {id: e.id, text: e.text, state: {checked: e.states.checked, expanded: e.states.expanded}}
      this.list = this.list.map((i) => {
        if (i.id === item.id) {
          return {
            ...i,
            ...item
          }
        } else return i
      })
      if (update) {
        this.updateData()
      }
      this.$emit("save", {list: this.list, text: this.text})
    },
    addParsing(data, parent_id = 0) {
      this.input = "";
      let arr = data.split('\n')
      let newItem = {}
      for (let value of arr) {
        if (value.length) {
          let maxId = maxBy(this.list, function (o) {
            return o.id;
          })
          newItem = {
            id: maxId ? maxId.id + 1 : 1,
            text: value,
            parent_id: parent_id,
            state: {checked: false, expanded: false}
          }
          this.list.push(newItem);
          this.updateData()
        }
      }
      this.data2.html_options.totalInfo = this.list.length
      this.$emit("save", {list: this.list, text: this.text})
      return newItem
    },
    async remove(e) {
      const res = await this.$dialog.confirm({
        title: "Удалить",
        text: e.text + " ?",
        persistent: false,
      });
      if (!res) return;
      this.list = this.list.filter((i) => i.id !== e.id)
      this.data2.html_options.totalInfo = this.list.length
      this.updateData()
      this.$emit('save', {list: this.list, text: this.text})
    },
    dragging(targetNode, destinationNode, position) {

      let item = {id: targetNode.id}
      if (targetNode.parent) {
        item.parent_id = targetNode.parent.id
      } else item.parent_id = 0
      let newItem = this.list.find((i) => i.id === item.id)
      newItem.parent_id = item.parent_id
      let copyList = cloneDeep(this.list)
      copyList = copyList.filter((i) => i.id !== item.id)
      let newIndexItem
      if (position === "drag-on") {
        if (destinationNode.children.length > 1) {
          let indexFrontItem = destinationNode.children.length - 2
          let idFrontItem = destinationNode.children[indexFrontItem].id
          let indexBeside = copyList.findIndex((i) => i.id === idFrontItem)
          newIndexItem = indexBeside + 1
        }
        // this.$refs.tree.tree.expand(destinationNode)

        copyList = copyList.map((i) => {
          if (i.id === destinationNode.id) {
            i.state.expanded = true
            return i
          } else return i
        })

      } else {
        let besideItemId = destinationNode.id
        let indexBeside = copyList.findIndex((i) => i.id === besideItemId)
        if (position === 'drag-above') {
          if (indexBeside === 0) {
            newIndexItem = 0
          } else newIndexItem = indexBeside - 1
        }
        if (position === 'drag-below') newIndexItem = indexBeside + 1
      }
      copyList.splice(newIndexItem, 0, newItem)
      this.list = copyList
      this.updateData()
      this.$emit('save', {list: copyList, text: this.text})
    }
  },


}

</script>
<style lang="scss" scoped>
.pseudo-checkbox {
  margin-left: 22px;
  margin-right: 10px;
  width: 15px;
  height: 14px;
  box-sizing: border-box;
  border-radius: 2px;
  border: 1px solid #aaa;
  background: none;
}

::v-deep.custom {
  .l-fade-enter-active, .l-fade-leave-active {
    transition: none;
    transform: none;
  }

  .l-fade-enter, .l-fade-leave-to {
    opacity: 0;
    transform: none;
  }

  .tree-node.selected > .tree-content {
    background-color: inherit;
  }

  .tree-node.selected > .tree-content:hover {
    background: #f6f8fb;
  }

  .tree-dragnode {
    display: none;
  }

  .tree {
    font-size: 14px;
  }

  //.tree-arrow::before {
  //  content: '\205E\205E';
  //  margin-left: -20px;
  //  color: #c2c2c2;
  //  font-weight: 500;
  //}
  .tree-arrow {
    height: 20px;
    margin-left: 28px;
    margin-top: -7px;
  }

  .tree-arrow.has-child {
    width: 16px;
    margin-left: 12px;
    z-index: 3;
  }

  .tree-arrow.has-child:after {
    height: 6px;
    width: 6px;
    margin-left: -1px;
    margin-top: 3px;
  }

  .tree-checkbox {
    width: 14px;
    height: 14px;
    margin-left: 3px;
    top: 1px;
    border: 1px solid #aaa;
    background: none;
  }

  .tree-checkbox.checked {
    background-color: #3a99fc;
  }

  .tree-checkbox.checked:after {
    left: 4px;
    top: 0;
    height: 8px;
    width: 2.5px;
    border: 2px solid #fff;
    border-left: 0;
    border-top: 0;
  }

  .tree-content {
    padding: 0 !important;
    align-items: baseline;

    &:hover {
      .todo-btn-delete {
        opacity: 0.5;
      }
    }
  }

  .drag-above > .tree-content::before, .drag-below > .tree-content::after {
    display: block;
    content: '';
    position: absolute;
    height: 8px;
    left: 0;
    right: 0;
    z-index: 2;
    box-sizing: border-box;
    background-color: #F5B75AFF;
    border: 3px solid #F5B75AFF;
    background-clip: padding-box;
    border-bottom-color: transparent;
    border-top-color: transparent;
    border-radius: 0;
  }

  .drag-on > .tree-content {
    background: #fffde0 !important;
    outline: 1px solid #F5B75AFF;
  }

  //.drag-above > .tree-content::before, .drag-below > .tree-content::after {
  //  display: block;
  //  content: '';
  //  position: absolute;
  //  height: 8px;
  //  left: 0;
  //  right: 0;
  //  z-index: 2;
  //  box-sizing: border-box;
  //  background-color: #afafaf;
  //  border: 3px solid #afafaf;
  //  background-clip: padding-box;
  //  border-bottom-color: transparent;
  //  border-top-color: transparent;
  //  border-radius: 0;
  //}
  //
  //.drag-on > .tree-content {
  //  background: #fffde0!important;
  //  outline: 1px solid #afafaf;
  //}


  .tree-children {
    margin-left: -9px;
  }

  .tree-anchor {
    padding: 0 6px;

    div {
      width: 100%;

      div {
        width: auto;
      }
    }
  }

  .tree-input {
    border: none;
    border-bottom: 1px solid #1b83d8;
    margin: 2px 0;
    font-size: 14px;
    width: 95%;
  }
}

.todo-list {
  position: relative;

  overflow-y: auto;
  padding-right: 0px;

  ::-webkit-scrollbar-track {
    background-color: none;
  }

  ::-webkit-scrollbar {
    width: 7px;
    height: 7px;
    background-color: none;
  }

  ::-webkit-scrollbar-thumb {
    background-color: rgb(184, 184, 184);
  }

  .item {
    &:hover {
      .todo-btn-delete {
        opacity: 0.5;
      }
    }
  }
}

.not-list {
  text-align: center;
  color: #cccccc;
  font-style: italic;
  margin-bottom: 16px;
}

::v-deep.input-custom {
  border: 1px solid rgba(255, 255, 255, 0);
  border-bottom: 1px solid #aaa;

  &:hover {
    border: 1px solid #aaa;
    border-radius: 3px;
    margin-top: 2px;
  }


  font-size: 13px !important;

  .v-input__slot {
    min-height: 25px !important;
  }

  .v-input__slot:hover {
    border: none !important;
  }

  .v-input__slot:before {
    display: none;
  }

  .v-input__slot:after {
    display: none !important;
  }

  .v-input__control {
    min-height: 25px
  }

  textarea {
    padding-top: 3px;
    min-height: 25px;
    margin-top: 0 !important;
    line-height: 1.5 !important;
    padding-left: 5px;

    &:focus::placeholder {
      color: transparent;
    }
  }

  .v-input__append-inner {
    margin-top: 0 !important;

    button {
      color: #666666 !important;
    }
  }

}

.card-tabs {
  font-size: 11px;
  display: flex;
  border-bottom: 1px solid #e0e0e0;

  .item {
    display: flex;
    align-items: center;
    cursor: pointer;

    &:last-child {
      margin-right: 0 !important;
    }
  }

  .tabs-title {
    max-width: 130px;
    overflow: hidden;
    text-overflow: ellipsis;
    text-transform: uppercase;
    white-space: nowrap;
    opacity: 0.5;
  }

  .active {
    position: relative;

    &:before {
      content: "";
      position: absolute;
      bottom: -5px;
      right: 0;
      left: 0;
      background: #2196f3;
      height: 1px;
    }

    .tabs-title {
      opacity: 1;
    }
  }

  .icon {
    width: 15px;
    height: 15px;

    img {
      width: 15px;
      height: 15px;
    }
  }
}

</style>
