<template>
<div v-cloak>

  <v-tooltip  v-if="show_re_open_button"
              left>
    Edit UI Schema
    <template v-slot:activator="{ on }">
      <v-btn
        style="position: absolute; top: 25px; right: 25px"
        color="primary"
        data-cy="reopen_ui_schema"
        @click="$emit('start_edit_ui_schema')"
        fab
        right
        absolute
        v-on="on"
      >
        <v-icon> mdi-puzzle-edit-outline</v-icon>
      </v-btn>
    </template>
  </v-tooltip>

  <div
    class="context-menu"
    :class="{visible: show_context_menu}"
    :style="{top: top, left: left}"
  >

    <v-card
      class="mx-auto"
      max-width="300"
      tile
    >

      <ui_schema_menu_content
        :schema_id="schema_id"
        @hide="hide"
      >
      </ui_schema_menu_content>
    </v-card>

  </div>

  <div
      class="save-menu"
      :class="{visible: show_context_menu}"
      :style="{top: '270px', right: '0px', zIndex: 1}"
    >

    <v-card outlined
            shaped
            elevation="5"
            >
      <v-card-title class="pb-0">
        <tooltip_button
          v-if="!edit_name"
          tooltip_message="Edit Name"
          datacy="ui_schema_edit_name"
          @click="edit_name = !edit_name"
          icon="mdi-rename-box"
          :icon_style="true"
          color="primary"
          :disabled="!$store.state.ui_schema.current"
        >
        </tooltip_button>
        <tooltip_button
          v-else
          tooltip_message="Stop Editing"
          datacy="ui_schema_edit_name"
          @click="edit_name = !edit_name"
          icon="mdi-cancel"
          :icon_style="true"
          color="primary"
          :disabled="!$store.state.ui_schema.current"
        >
        </tooltip_button>
        <div v-if="edit_name == false" @dblclick="edit_name=true">
          {{$store.state.ui_schema.current.name}}
        </div>

        <div class="pb-0">
          <v-text-field
            v-if="edit_name == true"
            v-model="$store.state.ui_schema.current.name"
            @input="has_changes = true"
            @focus="$store.commit('set_user_is_typing_or_menu_open', true)"
            @blur="$store.commit('set_user_is_typing_or_menu_open', false)"
            label="Schema Name:"
            flat
              >
          </v-text-field>
        </div>


      </v-card-title>

      <v-alert
            type="info"
            dismissible>
        <b>Editing UI Design</b> <br>
        Hover over a button to show options. Click plus to add buttons.
      </v-alert>

      <v-container>

        <v-layout>
          <tooltip_button
              tooltip_message="Add"
              datacy="ui_schema_add"
              button_message="Add Elements"
              @click="show_add_menu=true"
              icon="add"
              :text_style="true"
              color="primary"
                          >
          </tooltip_button>

          <tooltip_button
            v-if="show_save"
            datacy="ui_schema_save"
            tooltip_message="Save"
            @click="update_ui_schema_with_servercall()"
            icon="save"
            :loading="loading"
            :disabled="loading
                    || !ui_schema_exists
                    || public_not_super_admin
                    || !current_ui_schema"
            :icon_style="true"
            color="primary"
                        >
        </tooltip_button>

        <tooltip_button
            v-if="!$store.state.ui_schema.current.archived"
            tooltip_message="Archive"
            @click="toggle_is_archive()"
            icon="archive"
            :icon_style="true"
            color="primary"
            :disabled="!ui_schema_exists
                      || public_not_super_admin"
                        >
        </tooltip_button>


        <tooltip_button
            v-if="$store.state.ui_schema.current.archived"
            tooltip_message="Restore"
            @click="toggle_is_archive()"
            icon="mdi-delete-restore"
            :icon_style="true"
            color="primary"                          >
        </tooltip_button>


        <tooltip_button
            tooltip_message="Restore Defaults"
            @click="reset()"
            datacy="reset_defaults"
            icon="mdi-restore "
            :icon_style="true"
            color="primary"                          >
        </tooltip_button>

        <tooltip_button
            v-if="$store.state.user.current.is_super_admin == true"
            tooltip_message="Toggle is Public Example"
            @click="toggle_is_public()"
            icon="mdi-earth"
            :icon_style="true"
            color="primary"
                        >
        </tooltip_button>

          <div  v-if="$store.state.user.current.is_super_admin == true">
              <div v-if="$store.state.ui_schema.current.is_public">
                  Public
              </div> <div v-else>
                  Not Public
              </div>
          </div>

        </v-layout>

      <v-divider class="pb-2 pt-2"></v-divider>

      <v-layout>

        <v_error_multiple :error="error">
        </v_error_multiple>


        <ui_schema_selector
          ref="ui_schema_selector"
          :project_string_id="project_string_id"
          @change="change($event)"
          :disabled="selector_disabled"
          :current_ui_schema_prop="newly_created_schema"
                        >
        </ui_schema_selector>


        <tooltip_button
            tooltip_message="New Schema"
            datacy="ui_schema_new"
            @click="new_ui_schema_with_servercall()"
            icon="add"
            :icon_style="true"
            color="primary"
                        >
        </tooltip_button>

        <v-menu
        v-model="show_add_menu"
        :allow-overflow="true"
        :offset-overflow="true"
        :close-on-click="false"
        :close-on-content-click="false"
        :attach="true"
        :z-index="999999"
      >
        <v-card>
          <v-card-title>Add</v-card-title>
          <v-card-text>

            <diffgram_select
                :item_list="buttons_list_available"
                v-model="button_to_add"
                @focus="refresh_buttons_list_available()"
                label="Buttons"
                >
            </diffgram_select>

          </v-card-text>
          <v-card-actions>
            <v-btn @click="show_add_menu = false">Close
            </v-btn>
            <v-btn data-cy="add_selected"
                   @click="add_selected"
                   color="success">
              Add Selected
            </v-btn>
          </v-card-actions>
        </v-card>
      </v-menu>


        <tooltip_button
            tooltip_message="Copy to New"
            datacy="ui_schema_copy"
            @click="copy_ui_schema_with_servercall()"
            icon="mdi-content-copy"
            :icon_style="true"
            color="primary"
            :disabled="!ui_schema_exists"
                        >
        </tooltip_button>


      </v-layout>

    <v-layout>

    <v-divider class="pb-2 pt-2"></v-divider>

    <v-card-actions>
      <v-btn
        color="red"
        text
        @click="close()"
      >
        Exit
      </v-btn>
    </v-card-actions>

    </v-layout>

  </v-container>
</v-card>
</div>
<v-snackbar  timeout="2000" top v-if="snackbar_visible" color="success" v-model="snackbar_visible">
  <h4 class="text-center">{{snackbar_message}}</h4>
</v-snackbar>
</div>

</template>

<script lang="ts">

  import Vue from 'vue';
  import axios from '../../services/customInstance';
  import {UI_Schema} from './ui_schema'
  import ui_schema_selector from './ui_schema_selector'
  import ui_schema_menu_content from './ui_schema_menu_content'

  export default Vue.extend({
    name: 'UISchemaContextMenu',
    components: {
      ui_schema_selector,
      ui_schema_menu_content,
    },
    props: {
      'mouse_position': {
        type: Object,
        default: null,
      },
      'project_string_id':{
        type: String,
        default: null
      },
      'schema_id':{
        required: true
      },
      'show_context_menu':{
        type: Boolean,
        default: false
      },
      'show_save' :{
        default: true
      },
      'create_new_on_load' :{
        default: false
      },
      'label_settings': {
        default: null
      }

    },
    data() {
      // move context menu off the page out of view when hidden
      return {
        selector_disabled: false,
        snackbar_visible: false,
        snackbar_message: "",
        top: '-1000px',
        left: '-1000px',
        instance_hover_index_locked: null,
        show_share_instance_menu: false,
        locked_mouse_position: undefined,
        show_add_menu: false,
        show_schema_editing_snackbar: true,

        edit_name: false,
        loading: false,
        error: {},

        show_re_open_button: false,

        newly_created_schema: undefined,

        button_to_add: undefined,

        buttons_list_original: [
            {'name': 'logo',
             'display_name': 'Logo',
             'image-icon': 'https://res-4.cloudinary.com/crunchbase-production/image/upload/c_lpad,h_25,w_25,f_auto,b_white,q_auto:eco/okhxici7vjqqznihxezz',
            },
            {'name': 'home',
             'icon': 'mdi-home',
             'display_name': 'Home',
            },
            {'name': 'task_list',
             'icon': 'mdi-playlist-play',
             'display_name': 'Task List',
            },
            {'name': 'undo',
             'icon': 'mdi-undo',
             'display_name': 'Undo',
            },
            {'name': 'redo',
             'icon': 'mdi-redo',
             'display_name': 'Redo',
            },
            {'name': 'defer',
             'icon': 'mdi-debug-step-over',
             'display_name': 'Defer',
            },
            {'name': 'zoom',
             'icon': 'mdi-magnify-plus-outline',
             'display_name': 'Zoom Display',
            },
            {'name': 'save',
             'icon': 'save',
             'display_name': 'Save',
            },
            {'name': 'label_selector',
             'icon': 'mdi-format-paint',
             'display_name': 'Label Selector',
            },
            {'name': 'instance_selector',
             'icon': 'mdi-vector-polygon',
             'display_name': 'Spatial Tool Selector',
            },
            {'name': 'previous_task',
             'display_name': 'Previous Task',
             'icon': 'mdi-chevron-left-circle',
             'color': 'primary'
            },
            {'name': 'next_task',
             'display_name': 'Next Task',
             'icon': 'mdi-chevron-right-circle',
             'color': 'primary'
            },
            {'name': 'edit_instance_template',
             'display_name': 'Edit Instance Spatial Template',
             'icon': 'mdi-vector-polyline-edit',
             'color': 'primary'
            },
            {'name': 'draw_edit',
             'display_name': 'Draw Edit Toggle',
             'icon': 'mdi-toggle-switch',
             'color': 'primary'
            },
            {'name': 'brightness_contrast_filters',
             'display_name': 'Brightness, Contrast, Filters',
             'icon': 'exposure',
             'color': 'primary'
            },
          {'name': 'time_tracking',
            'display_name': 'Time Tracking Widget',
            'icon': 'mdi-timer',
            'color': 'primary'
          },

          ],
        buttons_list_available: []
      }
    },

    computed: {
      current_ui_schema: function(){
        return this.$store.state.ui_schema.current;
      },
      public_not_super_admin: function () {
           if (this.$store.state.ui_schema.current.is_public == true
            && this.$store.state.user.current.is_super_admin != true) {
               return true
           }
       }

    },

    watch: {
      label_settings: {
        handler: function(new_val, old_val){
          this.$store.commit('set_ui_schema_element_value',
            ['label_settings', "default_settings", {...new_val}])
        },
        deep: true
      },
      show_context_menu(isVisible) {

        if (isVisible == true) {
          // lock mouse position on click only
          this.get_mouse_position();
          this.instance_hover_index_locked = this.instance_hover_index

          // We want to free condition on null in template so "normalize" it here.
          if (isNaN(this.instance_hover_index_locked)) {
            this.instance_hover_index_locked = null
          }

          this.show_re_open_button = false

        } else {

          this.top = '-1000px';
          this.left = '-1000px';
          this.instance_hover_index_locked = null
        }
      },
    },
    created() {
    },
    mounted() {

      if (this.$route.query.create_new_on_load == 'true'
        || this.$props.create_new_on_load) {
        this.new_ui_schema_with_servercall()
      }

      var self = this
      this.get_target_element_watcher = this.$store.watch((state) => {
          return this.$store.state.ui_schema.target_element
        },
        (new_val, old_val) => {
          self.get_mouse_position()
        },
      )
      this.show_ui_schema_add_menu = this.$store.watch((state) => {
          return this.$store.state.ui_schema.ui_schema_add_menu
        },
        (new_val, old_val) => {
          this.show_add_menu = new_val
        },
      )
      this.show_ui_schema_refresh = this.$store.watch((state) => {
          return this.$store.state.ui_schema.refresh
        },
        (new_val, old_val) => {

        },
      )
    },
    beforeDestroy() {
      this.get_target_element_watcher()
      this.show_ui_schema_add_menu()
      this.show_ui_schema_refresh()
    },
    methods: {

      ui_schema_exists: function () {
        if (this.get_ui_schema().client_created_time ||   // frontend
            this.get_ui_schema().id) { // backend
            return true
        }
        return false
      },

      refresh_buttons_list_available: function () {
        let list = []
        for (var button of this.buttons_list_original) {
          if (this.$store.getters.get_ui_schema(button.name, 'visible') == false) {
            list.push(button)
          }
        }
        this.buttons_list_available = list
      },
      get_ui_schema: function () {
        if (this.$store.state.ui_schema.current == undefined) {
          throw new Error("this.$store.state.ui_schema.current is undefined")
        }
        return this.$store.state.ui_schema.current
      },
      get_target_element: function () {
        // careful target is stored on ui_schema generally not `current`
        return this.$store.state.ui_schema.target_element
      },
      get_mouse_position: function () {
        if (!this.$store.state.ui_schema.target_element) {
          // we don't update mouse position unless exiting the edit mode
          this.show_add_menu = false
          return
        }
        let event = this.$store.state.ui_schema.event
        this.top = event.clientY - event.offsetY + 'px';
        this.left = event.clientX - event.offsetX + 'px';
        //this.locked_mouse_position = {...this.mouse_position};
      },
      close(){
        this.show_re_open_button = true
        this.$emit('close_context_menu')
      },
      reset(){
        this.$store.commit('reset_ui_schema')
      },
      hide() {
        this.$store.commit('set_ui_schema_element_value',
          [this.get_target_element(),'visible', false])
        //this.close();
      },
      show() {
        this.$store.commit('set_ui_schema_element_value',
          [this.get_target_element(),'visible', true])
        //this.close();
      },
      add_selected() {
        this.$store.commit('set_ui_schema_element_value',
          [this.button_to_add,'visible', true])
        this.button_to_add = undefined
      },

      new_ui_schema_with_servercall: async function(){


        let ui_schema = new UI_Schema
        ui_schema.new()

        this.loading = true;
        this.error = {}

        try{
          const result = await axios.post(
            `/api/v1/project/${this.project_string_id}/ui_schema/new`,
            ui_schema.serialize()
          )
          if(result.status === 200){

            this.change(result.data.ui_schema)
            this.edit_name = true // assume a user wants to edit name of new script
            this.newly_created_schema = result.data.ui_schema
          }

        }
        catch (error) {
          this.error = this.$route_api_errors(error)
          console.error(error)

        }
        finally {
          this.loading = false;
        }

      },
      show_snackbar: function(message){
        this.snackbar_visible = true;
        this.snackbar_message = message
      },
      update_ui_schema_with_servercall: async function(){
        if (!this.get_ui_schema() || !this.get_ui_schema().id) {
          return
        }

        this.loading = true;
        this.error = {}

        try{
          const response = await axios.post(
            `/api/v1/project/${this.project_string_id}/ui_schema/update`,
            this.get_ui_schema()
          )
          if(response.status === 200){
            await this.$refs.ui_schema_selector.refresh();
            this.$refs.ui_schema_selector.set_ui_schema(response.data.ui_schema);
            this.show_snackbar("UI Schema Saved.")
          }

        }
        catch (error) {
          this.error = this.$route_api_errors(error)
          console.error(error)

        }
        finally {
          this.loading = false;
        }

      },

      change: function (event) {
        if(!event) { return }
        if(event.id == this.$store.state.ui_schema.current.id) { return }

        this.$store.commit('set_ui_schema', event)
        this.$emit('set_ui_schema', event)
      },

      toggle_is_visible: async function () {

        this.$store.commit('set_ui_schema_top_level_key_value',
          ['is_visible', !this.get_ui_schema().is_visible])
        this.update_ui_schema_with_servercall()

      },

      toggle_is_archive: async function () {

        this.$store.commit('set_ui_schema_top_level_key_value',
          ['archived', !this.get_ui_schema().archived])
        this.update_ui_schema_with_servercall()

      },

      toggle_is_public: async function () {

        // requires super admin
        this.$store.commit('set_ui_schema_top_level_key_value',
          ['is_public', !this.get_ui_schema().is_public])
        this.update_ui_schema_with_servercall()

      },

      copy_ui_schema_with_servercall: async function(){

        let ui_schema = new UI_Schema
        const new_ui_schema = ui_schema.copy(this.get_ui_schema())

        this.loading = true;
        this.error = {}

        try{
          const result = await axios.post(
            `/api/v1/project/${this.project_string_id}/userscript/new`,
            new_ui_schema
          )
          if(result.status === 200){

            this.change(result.data.ui_schema)
            this.edit_name = true // assume a user wants to edit name of new script
          }
        }
        catch (error) {
          this.error = this.$route_api_errors(error)
          console.error(error)
        }
        finally {
          this.loading = false;
        }
      },


    }
  });
</script>

<style>
  .context-menu {
    position: absolute;
    margin: 0;
    box-sizing: border-box;
    display: none;
    z-index: 10000;
  }

  .save-menu {
    position: absolute;
    margin: 0;
    box-sizing: border-box;
    display: none;
    z-index: 1000;
  }

  .context-menu.visible {
    display: block;
  }
  .save-menu.visible {
    display: block;
  }
</style>
