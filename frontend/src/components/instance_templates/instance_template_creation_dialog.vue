<template>
  <v-dialog v-model="is_open" width="700px" :persistent="true" :no-click-animation="true"
            content-class="dialog-instance-template">

    <v-card elevation="0" class="pa-4 ma-0 pointertest">
      <v-card-title>
        Create KeyPoints Template:

      </v-card-title>
      <v-card-text>
        <v-container class="flex flex-column pa-0 ma-0" >
          <v-text-field class="ma-0 pa-0"
                        data-cy="instance_template_name_text_field"
                        label="Name"
                        v-model="name">

          </v-text-field>
          <v-alert
            dismissible
            color="secondary"
            text icon="mdi-information"
            class="ma-0 pa-0 pr-2 pl-2">
            Right Click on a Point to Name it, or Set Default Occlusion Value.
            Press Esc to stop drawing and go to edit mode.
            Double click a point to delete it.
          </v-alert>
          <instance_template_creation_toolbar
            class="ma-0 pa-0"
            ref="instance_template_creation_toolbar"
            :color_tool_active="instance_context.color_tool_active"
            @draw_mode_update="update_draw_mode_on_instances"
            @set_background="set_background"
            :project_string_id="project_string_id"
            :instance="instance"
            @zoom_in="zoom_in"
            @zoom_out="zoom_out"
            @mode_set="on_mode_set"
            @change_color="change_color"
            @coloring_tool_clicked="activate_color_tool"
          >

          </instance_template_creation_toolbar>

          <v_error_multiple :error="error">
          </v_error_multiple>
          <drawable_canvas
            @mousemove="mouse_move"
            @mousedown="mouse_down"
            @dblclick="double_click"
            @mouseup="mouse_up"
            @contextmenu="contextmenu"
            :canvas_width="canvas_width"
            :show_target_reticle="instance_context.draw_mode && instance && !instance.is_drawing_edge && !instance_context.color_tool_active"
            :show_context_menu="show_context_menu"
            :canvas_height="canvas_height"
            :reticle_colour="{
              hex: '#f0f0f0',
              rgba: {
                r: 255,
                g: 0,
                b: 0,
                a: 1
              }
            }"
            :image_bg="image_bg"
            :annotations_loading="false"
            :bg_color="bg_color"
            :auto_scale_bg="true"
            ref="instance_template_canvas"
          >
            <instance_drawer
              slot="instance_drawer"
              @instance_hover_update="instance_hover_update($event[0], $event[1])"
              :instance_list="instance_list"
            ></instance_drawer>
            <context_menu_instance_template
              slot="context_menu"
              slot-scope="props"
              :show_context_menu="show_context_menu"
              :project_string_id="project_string_id"
              :mouse_position="props.mouse_position"
              :instance="instance"
              @hide_context_menu="hide_context_menu"


            />

          </drawable_canvas>

        </v-container>

      </v-card-text>

      <v-card-actions class="flex justify-end pa-0">
        <v-btn color="error"
               text
               @click="is_open = false"
               :disabled="loading"
        >
          <v-icon>mdi-close</v-icon>
          Discard Changes
        </v-btn>
        <v-btn color="success"
               data-cy="save_instance_template_button"
               text
               @click="save_instance_template"
               :disabled="loading">
          <v-icon>mdi-content-save</v-icon>
          Save Instance Template
        </v-btn>
      </v-card-actions>

    </v-card>
    <v-snackbar color="primary"
                :timeout="5000"
                v-if="show_snackbar"
                v-model="show_snackbar"
                :multi-line="true"
                bottom
    >
      {{ snackbar_text }}
    </v-snackbar>
  </v-dialog>
</template>

<script>
  import Vue from "vue";
  import {KeypointInstance} from '../vue_canvas/instances/KeypointInstance';
  import context_menu_instance_template from '../context_menu/context_menu_instance_template';
  import {InstanceTemplateCreationInteractionGenerator} from '../vue_canvas/interactions/InstanceTemplateCreationInteractionGenerator';
  import drawable_canvas from '../vue_canvas/drawable_canvas';
  import axios from '../../services/customInstance';
  import instance_drawer from '../vue_canvas/instance_drawer';
  import instance_template_creation_toolbar from './instance_template_creation_toolbar'
  import {InstanceContext} from "../vue_canvas/instances/InstanceContext";
  import {iconFillPaint} from "@/utils/custom_icons";

  export default Vue.extend({
  name: "instance_template_creation_dialog",
  props: {
    project_string_id: undefined,
    instance_template: undefined,
    schema_id: {
      required: true
    },
  },
  components: {
    drawable_canvas: drawable_canvas,
    instance_drawer: instance_drawer,
    instance_template_creation_toolbar: instance_template_creation_toolbar,
    context_menu_instance_template: context_menu_instance_template,
  },
  data: function () {
    return {
      instance_type: 'keypoints',
      snackbar_text: null,
      instance_context: new InstanceContext(),
      draw_mode: true,
      show_hover_menu: false,
      show_snackbar: false,
      lock_point_hover_change: false,
      show_context_menu: false,
      instance: undefined,
      error: {},
      bg_color: 'grey',
      mode: '1_click',
      image_bg: undefined,
      canvas_wrapper: undefined,
      canvas_width: 600,
      canvas_height: 600,
      is_open: false,
      loading: false,
      name: undefined,
      label_settings: {
        show_occluded_keypoints: true,
        show_left_right_arrows: true
      }
    }
  },
  mounted() {
    if (window.Cypress) {
      window.InstanceTemplateDialog = this;
    }
    document.addEventListener('mousedown', this.mouse_events_global_down)
    this.instance_context.color = {
      hex: '#194d33',
      hsl: { h: 150, s: 0.5, l: 0.2, a: 1 },
      hsv: { h: 150, s: 0.66, v: 0.30, a: 1 },
      rgba: { r: 25, g: 77, b: 51, a: 1 },
      a: 1
    };

  },
  beforeDestroy() {
    document.removeEventListener('mousedown', this.mouse_events_global_down)
  },

  methods: {
    on_mode_set: function(mode){
      this.mode = mode;
    },
    change_color: function(color){
      console.log('asadasd', color)
      this.instance_context.color = color;
    },
    activate_color_tool: function(){
      if(this.instance_context.color_tool_active){
        this.instance_context.color_tool_active = false;
        this.close_snackbar()
      }
      else{
        this.instance_context.color_tool_active = true;
        this.open_snackbar('Click the nodes and edges to color them.')
      }

    },
    detect_clicks_outside_context_menu: function (e) {

      // skip clicks on the actual context menu
      if (e.target.matches('.context-menu, .context-menu *')) {
        return;
      }
      // assume if not on context menu, then it's outside and we want to hide it
      this.hide_context_menu()
    },
    mouse_events_global_down: function (e) {

      this.detect_clicks_outside_context_menu(e)

    },
    mouse_up_limits: function (event) {
      // 1: left, 2: middle, 3: right, could be null
      // https://stackoverflow.com/questions/1206203/how-to-distinguish-between-left-and-right-mouse-click-with-jquery
      if (event.which == 2 || event.which == 3) {
        return false
      }
      if (this.show_context_menu == true) {
        return false
      }

      return true

    },
    mouse_down_limits: function (event) {
      // 1: left, 2: middle, 3: right, could be null
      // https://stackoverflow.com/questions/1206203/how-to-distinguish-between-left-and-right-mouse-click-with-jquery
      if (event.which == 2 || event.which == 3) {
        return false
      }
      if (this.show_context_menu == true) {
        if (!this.instance.is_node_hovered) {
          this.hide_context_menu()
        }
        return false
      }
      return true

    },
    hide_context_menu: function () {
      this.show_context_menu = false;
    },
    open_context_menu: function () {
      this.show_context_menu = true;
    },
    open_snackbar: function (msg) {
      this.snackbar_text = msg;
      this.show_snackbar = true
    },
    close_snackbar: function () {
      this.snackbar_text = '';
      this.show_snackbar = false
    },
    open: async function () {
      this.is_open = true;
      await this.$nextTick();
      let nodes = [];
      let edges = [];
      this.instance = new KeypointInstance(
        this.$refs.instance_template_canvas.mouse_position,
        this.$refs.instance_template_canvas.canvas_ctx,
        this.instance_context,
        () => {
        },
        () => {
        },
        () => {
        },
        this.$refs.instance_template_canvas.mouse_down_delta_event,
        this.label_settings
      );


      // Set this to allow the creation of new nodes and edges.
      this.instance.template_creation_mode = true;

      if (this.$props.instance_template) {
        for (let i = 0; i < this.instance_template.instance_list.length; i++) {
          nodes = this.$props.instance_template.instance_list[i].nodes;
          edges = this.$props.instance_template.instance_list[i].edges;
          this.name = this.$props.instance_template.name;
          this.instance.nodes = nodes;
          this.instance.nodes.sort((a, b) => a.ordinal < b.ordinal ? -1 : 1)
          this.instance.edges = edges;
          this.instance.id = this.$props.instance_template.instance_list[i].id
          this.mode = this.$props.instance_template.mode;
          this.$refs.instance_template_creation_toolbar.set_mode(this.mode)
        }

      }
      window.addEventListener('keydown', this.key_down_handler);

    },
    zoom_in: function () {
      this.$refs.instance_template_canvas.zoom_in();
    },
    zoom_out: function () {
      this.$refs.instance_template_canvas.zoom_out();
    },
    set_background: function (image) {
      this.image_bg = image;
      this.bg_color = undefined
      let canvas = this.$refs.instance_template_canvas.canvas_ctx.canvas

    },
    reset_drawing: function () {
      // TODO: this can become an interaction in the future.
      if (this.instance.type === 'keypoints') {
        this.instance.is_drawing_edge = false;
        this.instance.is_moving = false;
        this.instance.is_node_hovered = false;
        this.instance.is_dragging_instance = false;
      }
    },
    key_down_handler: function (e) {

      if (e.keyCode === 27) {
        e.preventDefault();
        if (this.$refs['instance_template_creation_toolbar']) {
          this.$refs['instance_template_creation_toolbar'].draw_mode = !this.$refs['instance_template_creation_toolbar'].draw_mode;
          this.$refs['instance_template_creation_toolbar'].edit_mode_toggle();
          this.reset_drawing()
        }

      }
    },
    update_draw_mode_on_instances: function (draw_mode) {
      this.instance_context.draw_mode = draw_mode;
      this.instance_context.color_tool_active = false;
      if (draw_mode) {
        this.$refs.instance_template_canvas.canvas_element.style.cursor = 'none';
        this.instance.hovered_control_point_key = undefined;
      } else {
        this.$refs.instance_template_canvas.canvas_element.style.cursor = 'auto';
        this.instance.selected = false
        this.instance.is_drawing_edge = false;
      }

      if (this.instance_context.draw_mode) {
        this.open_snackbar('Press Esc to stop drawing Lines/Points and go to edit mode.');
      } else {
        this.close_snackbar()
      }
    },
    close: function () {
      window.removeEventListener('keydown', this.key_down_handler)
      this.is_open = false;
    },
    instance_hover_update(index, type) {
      if (this.lock_point_hover_change == true) {
        return
      }
      // important, we don't change the value if it's locked
      // otherwise it's easy for user to get "off" of the point they want

      if (index != null) {
        this.instance_hover_index = parseInt(index)
        this.instance_hover_type = type   // ie polygon, box, etc.
      } else {
        this.instance_hover_index = null;
        this.instance_hover_type = null;
      }
    },
    generate_interaction_from_event(event) {
      const interaction_generator = new InstanceTemplateCreationInteractionGenerator(
        event,
        this.instance_hover_index,
        this.instance_list,
        this.$refs['instance_template_creation_toolbar'].draw_mode,
        this.instance,
      )
      return interaction_generator.generate_interaction();
    },
    mouse_move: function (event) {
      if(this.instance_context.color_tool_active){
        this.instance.ctx.canvas.style.cursor =  `url(${iconFillPaint}), auto`;
      }
      const interaction = this.generate_interaction_from_event(event);
      if (interaction) {
        interaction.process();
      }
    },
    mouse_down: function (event) {
      if (!this.mouse_down_limits(event)) {
        return
      }

      const interaction = this.generate_interaction_from_event(event);
      if (interaction) {
        interaction.process();
      }

    },
    double_click: function (event) {

      if (!this.show_context_menu) {
        this.instance.double_click(event);
        this.hide_context_menu()
        this.$forceUpdate();
      }

    },
    mouse_up: function (event) {
      if (!this.mouse_up_limits(event)) {
        return
      }
      const interaction = this.generate_interaction_from_event(event);
      if (interaction) {
        interaction.process();
        this.$forceUpdate();
      }
    },
    contextmenu: function (event) {
      event.preventDefault()
      this.instance.contextmenu(event);
      this.open_context_menu();
    },
    validate_empty_instance_list() {
      // Checks if all the instances in the instance list are non-empty.
      let result = true;
      this.instance_list.forEach(inst => {
        // TODO VALIDATE OTHER INSTANCE TYPES
        if (inst.type === 'keypoints') {
          if (inst.nodes.length === 0 && inst.edges.length === 0) {
            this.error = {keypoints: 'Please add at least one point.'}
            result = false;
          }
        }
      })
      return result
    },
    save_instance_template: function () {
      if (this.$props.instance_template) {
        this.update_instance_template()
      } else {
        this.create_instance_template()
      }
    },
    update_instance_template: async function () {
      try {
        this.error = {};
        this.loading = true;
        const has_empty_instances = this.validate_empty_instance_list();
        if (!has_empty_instances) {
          return
        }
        if (!this.name) {
          this.error = {'name': 'Please provide a name for the instance template.'}
          return
        }
        if (!this.$props.instance_template) {
          return
        }

        const response = await axios.post(
          `/api/v1/project/${this.$props.project_string_id}/instance-template/${this.$props.instance_template.id}`,
          {
            name: this.name,
            mode: this.mode,
            instance_list: this.instance_list.map(inst => inst.get_instance_data()),

          })
        if (response.status === 200) {
          this.is_open = false;
          this.$emit('instance_template_create_success', response.data.instance_template);
        }
      } catch (error) {
        console.error(error)
        this.error = this.$route_api_errors(error)

      } finally {
        this.loading = false
      }
    },
    create_instance_template: async function () {
      try {
        this.error = {};
        const has_empty_instances = this.validate_empty_instance_list();
        if (!has_empty_instances) {
          return
        }
        if (!this.name) {
          this.error = {'name': 'Please provide a name for the instance template.'}
          return
        }

        const response = await axios.post(`/api/v1/project/${this.$props.project_string_id}/instance-template/new`,
          {
            name: this.name,
            reference_height: this.canvas_height,
            reference_width: this.canvas_width,
            schema_id: this.schema_id,
            mode: this.mode,
            instance_list: this.instance_list.map(inst => inst.get_instance_data()),

          })
        if (response.status === 200) {
          this.name = [];
          this.is_open = false;
          this.$emit('instance_template_create_success', response.data.instance_template);
        }
      } catch (error) {
        console.error(error)
        this.error = this.$route_api_errors(error)

      } finally {
        this.loading = false
      }
    }
  },

  computed: {
    instance_list() {
      if (this.instance) {
        return [this.instance]
      }
      return []
    }
  }

})
</script>

<style>
.dialog-instance-template {
  max-height: 100% !important;
  overflow: inherit !important;
}
</style>
