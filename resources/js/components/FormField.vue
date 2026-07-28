<template>
  <component
    :dusk="currentField.attribute"
    :is="currentField.fullWidth ? 'FullWidthField' : 'default-field'"
    :field="currentField"
    :errors="errors"
    :show-help-text="showHelpText"
    full-width-content
  >
    <template #field>
      <div ref="flexibleFieldContainer">
        <form-nova-flexible-content-group
          v-for="(group, groupIndex) in orderedGroups"
          :dusk="currentField.attribute + '-' + groupIndex"
          :key="group.key + '-' + reorderNonce"
          :field="currentField"
          :group="group"
          :index="groupIndex"
          :resource-name="resourceName"
          :resource-id="resourceId"
          :errors="errors"
          :mode="mode"
          @move-up="moveUp(group.key)"
          @move-down="moveDown(group.key)"
          @toggle-visibility="toggleVisibility(group.key)"
          @remove="remove(group.key)"
        />
      </div>

      <component
        :layouts="layouts"
        :is="currentField.menu.component"
        :field="currentField"
        :limit-counter="limitCounter"
        :limit-per-layout-counter="limitPerLayoutCounter"
        :errors="errors"
        :resource-name="resourceName"
        :resource-id="resourceId"
        @addGroup="addGroup($event)"
        @importGroup="importGroup"
      />

      <import-export-flexible-content-group-modal
        v-if="isImport"
        @close="isImport = false"
        :message="importMessage"
        ok="Ok"
        :name="groupName"
        title="Import group"
      />
    </template>
  </component>
</template>

<script>
import FullWidthField from "./FullWidthField";
import Sortable from "sortablejs";
import {
  DependentFormField,
  HandlesValidationErrors,
  mapProps,
} from "laravel-nova";
import Group from "../group";

export default {
  mixins: [HandlesValidationErrors, DependentFormField],

  props: {
    ...mapProps(["resourceName", "resourceId", "mode"]),
  },

  components: { FullWidthField },

  computed: {
    layouts() {
      return this.currentField.layouts || false;
    },
    orderedGroups() {
      return this.order.reduce((groups, key) => {
        groups.push(this.groups[key]);
        return groups;
      }, []);
    },

    limitCounter() {
      if (
        this.currentField.limit === null ||
        typeof this.currentField.limit == "undefined"
      ) {
        return null;
      }

      return this.currentField.limit - Object.keys(this.groups).length;
    },

    limitPerLayoutCounter() {
      return this.layouts.reduce((layoutCounts, layout) => {
        if (layout.limit === null) {
          layoutCounts[layout.name] = null;

          return layoutCounts;
        }

        let count = Object.values(this.groups).filter(
          (group) => group.name === layout.name,
        ).length;

        layoutCounts[layout.name] = layout.limit - count;

        return layoutCounts;
      }, {});
    },
  },

  data() {
    return {
      order: [],
      groups: {},
      files: {},
      sortableInstance: null,
      isImport: false,
      importMessage: null,
      groupName: null,
      // Bumped on every reorder and mixed into each group's :key so Vue remounts
      // the groups instead of moving their DOM nodes. Moving the DOM reloads
      // iframe-based fields (TinyMCE) and wipes their content; a remount lets the
      // fields re-initialise from their value instead (MARK-9137).
      reorderNonce: 0,
    };
  },

  beforeUnmount() {
    if (this.sortableInstance) {
      this.sortableInstance.destroy();
    }
  },

  methods: {
    /*
     * Set the initial, internal value for the field.
     */
    setInitialValue() {
      this.value = this.currentField.value || [];
      this.files = {};

      this.populateGroups();
      this.$nextTick(this.initSortable.bind(this));
    },

    /**
     * Fill the given FormData object with the field's internal value.
     */
    fill(formData) {
      let key, group;

      this.value = [];
      this.files = {};

      for (var i = 0; i < this.order.length; i++) {
        key = this.order[i];
        group = this.groups[key].serialize();

        // Only serialize the group's non-file attributes
        this.value.push({
          layout: group.layout,
          key: group.key,
          attributes: group.attributes,
          visibility: group.visibility,
        });

        // Attach the files for formData appending
        this.files = { ...this.files, ...group.files };
      }

      this.appendFieldAttribute(formData, this.currentField.attribute);
      formData.append(
        this.currentField.attribute,
        this.value.length ? JSON.stringify(this.value) : "",
      );

      // Append file uploads
      for (let file in this.files) {
        formData.append(file, this.files[file]);
      }

      this.$nextTick(this.initSortable.bind(this));
    },

    /**
     * Register given field attribute into the parsable flexible fields register
     */
    appendFieldAttribute(formData, attribute) {
      let registered = [];

      if (formData.has("___nova_flexible_content_fields")) {
        registered = JSON.parse(
          formData.get("___nova_flexible_content_fields"),
        );
      }

      registered.push(attribute);

      formData.set(
        "___nova_flexible_content_fields",
        JSON.stringify(registered),
      );
    },

    /**
     * Update the field's internal value.
     */
    handleChange(value) {
      this.value = value || [];
      this.files = {};

      this.populateGroups();
    },

    /**
     * Set the displayed layouts from the field's current value
     */
    populateGroups() {
      this.order.splice(0, this.order.length);
      this.groups = {};

      for (var i = 0; i < this.value.length; i++) {
        this.addGroup(
          this.getLayout(this.value[i].layout),
          this.value[i].attributes,
          this.value[i].key,
          this.currentField.collapsed,
          this.value[i].visibility,
        );
      }
    },

    /**
     * Retrieve layout definition from its name
     */
    getLayout(name) {
      if (!this.layouts) return;
      return this.layouts.find((layout) => layout.name == name);
    },

    /**
     * Append the given layout to flexible content's list
     */
    addGroup(layout, attributes, key, collapsed, visibility) {
      if (!layout) return;

      collapsed = collapsed || false;

      let fields = attributes || JSON.parse(JSON.stringify(layout.fields)),
        group = new Group(
          layout.name,
          layout.title,
          fields,
          this.currentField,
          key,
          collapsed,
          visibility,
        );

      this.groups[group.key] = group;
      this.order.push(group.key);
    },

    /**
     * Move a group up
     */
    moveUp(key) {
      let index = this.order.indexOf(key);

      if (index <= 0) return;

      this.order.splice(index - 1, 0, this.order.splice(index, 1)[0]);
      this.reorderNonce++;
    },

    /**
     * Move a group down
     */
    moveDown(key) {
      let index = this.order.indexOf(key);

      if (index < 0 || index >= this.order.length - 1) return;

      this.order.splice(index + 1, 0, this.order.splice(index, 1)[0]);
      this.reorderNonce++;
    },

    /**
     * Import a group from clipboard (sessionStorage)
     */
    importGroup() {
      try {
        const text = sessionStorage.getItem("exportImportGroup");

        if (!text) throw new Error("Nothing to import");

        let group = null;

        try {
          group = JSON.parse(text);
        } catch (error) {
          throw new Error("Imported data does not look like a content block");
        }

        if (!group || !group.key) throw new Error("Invalid data");

        this.groupName = group.title;

        const isAllowedToImport = !!this.layouts.find(
          (layout) => layout.name === group.name,
        );

        if (!isAllowedToImport)
          throw new Error("block cannot be imported to this page");

        // Give every nested flexible item a fresh key so the imported block's
        // field attributes (and thus TinyMCE editor ids) don't collide with the
        // block it was exported from — a duplicate id makes the copied editor
        // silently fail to initialise (MARK-9137).
        this.regenerateNestedFlexibleKeys(group);

        this.addGroup(group, null, null, group.collapsed);

        this.importMessage = "block has been successfully imported";
      } catch (error) {
        this.importMessage =
          error.message || "an error occured while importing the block";
      } finally {
        this.isImport = true;
      }
    },

    /**
     * Recursively assign a fresh key to every nested flexible-content item in an
     * imported payload. Flexible items are stored as { layout, key, attributes },
     * and their key drives the reconstructed field attributes (and TinyMCE editor
     * ids). Reusing the exported keys collides those ids with the source block.
     */
    regenerateNestedFlexibleKeys(node) {
      if (Array.isArray(node)) {
        node.forEach((item) => this.regenerateNestedFlexibleKeys(item));
        return;
      }

      if (!node || typeof node !== "object") {
        return;
      }

      if (
        typeof node.layout === "string" &&
        typeof node.key === "string" &&
        node.attributes &&
        typeof node.attributes === "object"
      ) {
        node.key = this.generateFlexibleKey();
      }

      Object.values(node).forEach((value) =>
        this.regenerateNestedFlexibleKeys(value),
      );
    },

    /**
     * Generate a unique flexible group key (mirrors Group.getTemporaryUniqueKey:
     * a "c" prefix to keep it a valid HTML id + 15 random chars).
     */
    generateFlexibleKey() {
      const charSet =
        "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789";
      let key = "";
      for (let i = 0; i < 15; i++) {
        key += charSet.charAt(Math.floor(Math.random() * charSet.length));
      }
      return "c" + key;
    },

    /**
     * Toggle a group's visibility
     */
    toggleVisibility(key) {
      this.groups[key].visibility = !this.groups[key].visibility;
    },

    /**
     * Remove a group
     */
    remove(key) {
      let index = this.order.indexOf(key);

      if (index < 0) return;

      this.order.splice(index, 1);
      delete this.groups[key];
    },

    initSortable() {
      const containerRef = this.$refs["flexibleFieldContainer"];

      if (!containerRef || this.sortableInstance) {
        return;
      }

      this.sortableInstance = Sortable.create(containerRef, {
        ghostClass: "nova-flexible-content-sortable-ghost",
        dragClass: "nova-flexible-content-sortable-drag",
        chosenClass: "nova-flexible-content-sortable-chosen",
        direction: "vertical",
        handle: ".nova-flexible-content-drag-button",
        scrollSpeed: 5,
        animation: 500,
        onEnd: (evt) => {
          const item = evt.item;
          const key = item.id;
          const oldIndex = evt.oldIndex;
          const newIndex = evt.newIndex;

          if (newIndex < oldIndex) {
            this.moveUp(key);
          } else if (newIndex > oldIndex) {
            this.moveDown(key);
          }
        },
      });
    },
  },
};
</script>
