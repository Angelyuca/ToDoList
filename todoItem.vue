<template>
  <div>
    <v-icon v-if="!activeNew" dense class="icon-drag todo-btn-delete">mdi-drag-vertical</v-icon>
    <div class="d-flex align-baseline" style="width: 100%">
      <div class="node-text" v-if="!edit"
           @click="viewImg"
           :inner-html.prop="item.text | linkPars"
           :class="{'done-input': item.states.checked}" @dblclick="activeEdit">
      </div>

      <miniEditor
          v-if="edit"
          class="custom-editor"
          :pasteHelper="pasteHelper"
          :value="item.text"
          @enter="editItem"
          v-click-outside="blurEdit"
          @clickElement="viewImg"
      />
      <div class="d-flex pl-3" style="width: auto">
        <!--        <v-btn-->
        <!--            small-->
        <!--            icon-->
        <!--            class="todo-btn-delete"-->
        <!--            @click.stop="activeEdit"-->
        <!--        >-->
        <!--          <v-icon-->
        <!--              small-->
        <!--          >mdi-square-edit-outline-->
        <!--          </v-icon>-->
        <!--        </v-btn>-->
        <!--        <v-btn-->
        <!--            small-->
        <!--            icon-->
        <!--            class="todo-btn-delete"-->
        <!--            @click.stop="activeNew = true"-->
        <!--            title="Добавить подпункт"-->
        <!--        >-->
        <!--          <v-icon-->
        <!--              small-->
        <!--          >mdi-playlist-plus-->
        <!--          </v-icon>-->
        <!--        </v-btn>-->
        <!--        <v-btn-->
        <!--            small-->
        <!--            icon-->
        <!--            class="todo-btn-delete"-->
        <!--            @click="$emit('remove', item)"-->
        <!--        >-->
        <!--          <v-icon-->
        <!--              small-->
        <!--          >mdi-delete-->
        <!--          </v-icon>-->
        <!--        </v-btn>-->

        <v-menu
            offset-y
            rounded="5"
            :close-on-content-click="true"
        >
          <template v-slot:activator="{ attrs, on }">
            <v-btn
                small
                icon
                v-bind="attrs"
                v-on="on"
                @click="active = true"
                class="todo-btn-delete"
            >
              <v-icon
                  small
              >mdi-dots-vertical
              </v-icon>
            </v-btn>
          </template>

          <v-list>
            <div
                v-for="item in actions"
            >
              <template v-if="item.text === 'divider'">
                <v-divider></v-divider>
              </template>

              <v-list-item
                  dense
                  v-else
                  link
              >
                <v-list-item-title class="text-btn" @click="actionsFunc(item.func)"
                                   v-text="item.text"></v-list-item-title>
              </v-list-item>
            </div>
          </v-list>
        </v-menu>


      </div>
    </div>
    <div class="new-sub" v-if="activeNew">
      <span class="pseudo-checkbox"></span>
      <miniEditor
          ref="editor"
          v-click-outside="blurSub"
          class="custom-editor"
          :pasteHelper="pasteHelper"
          :placeholder="$t('fields.new_subtask')"
          @enter="saveSub"
          @clickElement="viewImg"
      />
    </div>

  </div>
</template>

<script>
import miniEditor from "@/components/redactors/miniEditor/miniEditor";
import {Fancybox} from "@fancyapps/ui";

export default {
  name: "todoItem",
  components: {
    miniEditor,
  },
  props: ["item"],
  data() {
    return {
      showForm: false,
      active: false,
      edit: false,
      activeNew: false,
      textSub: '',
      actions: [
        {text: this.$t('btns.edit'), func: 'edit'},
        {text: this.$t('btns.add_subparagraph'), func: 'addSub'},
        {text: this.$t('btns.delete'), func: 'delete'},
        {text: 'divider', func: ''},
        {text: this.$t('fields.turn_into_task'), func: ''}]
    }
  },

  filters: {
    linkPars: function (value) {
      if (!value) return ''
      const urlRegex = /(?<!href\s*=\s*["']?)(https?:\/\/|ftp:\/\/|www\.)((?![.,?!;:()]*(\s|$))[^\s]){2,}/gi;
      return value.replace(urlRegex, function (url) {
        return '<a href="' + url + '" target="_blank">' + url + '</a>';
      })
    },
  },
  methods: {
    viewImg(e) {
      if (e.target.className === 'js-image' && e.target.dataset.hasOwnProperty('fancybox')) {
        e.preventDefault()
        Fancybox.show([
          {
            src: e.target.href,
            type: "image",
          },
        ])
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
    actionsFunc(action) {
      if (action === 'edit') this.activeEdit()
      if (action === 'addSub') this.activeNew = true
      if (action === 'delete') this.$emit('remove', this.item)
    },
    blurSub() {
      this.activeNew = false
      this.textSub = ''
      this.$refs.editor.clear()
    },
    blurEdit() {
      this.edit = false
    },
    saveSub(data) {
      this.$emit('addSub', {text: data, parent_id: this.item.id})
      this.activeNew = false
      this.textSub = ''
    },
    activeEdit() {
      this.edit = true
    },
    editItem(e) {
      let item = {
        ...this.item,
        text: e[0]
      }
      this.$emit("edit", item)
      this.edit = false
    }
  }
}
</script>

<style scoped lang="scss">
//.test{
//  background-color: #f6f8fb;
//}
::v-deep .v-menu {
  display: contents;
}

::v-deep .v-list-item--dense {
  min-height: 25px !important;
}

.text-btn {
  color: #7b7b7b;
  font-size: 12px;
}

.new-sub {
  display: flex;
  align-items: center;
  margin-left: -8px;
}

.v-input--checkbox {
  margin: 0;
  padding: 0;
}

.pseudo-checkbox {
  margin-right: 10px;
  width: 15px;
  height: 14px;
  border-radius: 2px;
  border: 1px solid #aaa;
  background: none;
}

.input-sub {
  border: none;
  border-bottom: 1px solid #aaa;
  margin: 0 0 4px 0;
  padding: 0;
  font-size: 14px;
  outline: none;
  width: 100%;
}

.custom-editor {
  width: 100% !important;
  margin-bottom: 3px;

  &:hover, &:focus,
  &.active {
    border-color: transparent;
    border-bottom: 1px solid #aaaaaa;
    border-radius: 0;
  }
}

.icon-drag {
  position: absolute;
  left: -2px;
  top: 4px
  //top: 50%;
  //transform: translateY(-50%);
}

.node-text {
  width: 100% !important;
  font-size: 14px;
  white-space: break-spaces;
  line-height: 1.4;
  min-width: 250px;
}

.done-input {
  text-decoration: line-through #cccccc;
  color: #cccccc;
}

.input-edit {
  margin-top: 0;
  margin-bottom: 3px;
}

::v-deep.v-input input {
  max-height: 25px !important;
}

.todo-btn-delete {
  opacity: 0;
  transition: all 0.3s ease;

  &:hover {
    opacity: 1 !important;
  }
}

</style>
