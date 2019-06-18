<template>
  <div class="interface-many-to-many">
    <v-notice v-if="relationshipSetup === false" color="warning" icon="warning">
      {{ $t("relationship_not_setup") }}
    </v-notice>

    <template v-else>
      <div v-if="items.length" class="table">
        <div class="header">
          <div class="row">
            <button v-if="sortable" class="sort-column" @click="toggleManualSort">
              <v-icon name="sort" size="18" :color="manualSortActive ? 'action' : 'light-gray'" />
            </button>
            <button
              v-for="field in visibleFields"
              :key="field.field"
              type="button"
              @click="changeSort(field.field)"
            >
              {{ $helpers.formatTitle(field.field) }}
              <v-icon
                v-if="sort.field === field.field"
                :name="sort.asc ? 'arrow_downward' : 'arrow_upward'"
                size="16"
              />
            </button>
          </div>
        </div>
        <draggable
          v-model="itemsSorted"
          class="body"
          handle=".drag-handle"
          :disabled="!sortable || !manualSortActive"
          :class="{ dragging }"
          @start="dragging = true"
          @end="dragging = false"
        >
          <div
            v-for="item in itemsSorted"
            :key="item[junctionRelatedKey][relatedPrimaryKeyField] || item.$tempKey"
            class="row"
            @click="startEdit(item[junctionPrimaryKey.field], item.$tempKey)"
          >
            <div v-if="sortable" class="sort-column" :class="{ disabled: !manualSortActive }">
              <v-icon name="drag_handle" class="drag-handle" />
            </div>
            <v-ext-display
              v-for="field in visibleFields"
              :key="field.field"
              :interface-type="field.interface"
              :name="field.field"
              :type="field.type"
              :collection="field.collection"
              :datatype="field.datatype"
              :options="field.options"
              :value="item[junctionRelatedKey][field.field]"
            />
          </div>
        </draggable>
      </div>

      <v-notice v-else>{{ $t("no_items_selected") }}</v-notice>

      <div class="buttons">
        <v-button
          v-if="options.allow_create"
          type="button"
          :disabled="readonly"
          icon="add"
          @click="addNew = true"
        >
          {{ $t("add_new") }}
        </v-button>

        <v-button
          v-if="options.allow_select"
          type="button"
          :disabled="readonly"
          icon="playlist_add"
          @click="selectExisting = true"
        >
          {{ $t("select_existing") }}
        </v-button>
      </div>
    </template>

    <v-item-select
      v-if="selectExisting"
      :fields="visibleFieldNames"
      :collection="relation.junction.collection_one.collection"
      :filters="[]"
      :value="stagedSelection || selectionPrimaryKeys"
      @input="stageSelection"
      @done="closeSelection"
      @cancel="cancelSelection"
    />

    <portal v-if="editExisting" to="modal">
      <v-modal
        :title="$t('editing_item')"
        :buttons="{
          save: {
            text: $t('save'),
            color: 'accent'
          }
        }"
        @close="editExisting = false"
        @save="saveEdits"
      >
        <div class="edit-modal-body">
          <v-form
            :fields="relation.junction.collection_one.fields"
            :collection="relation.junction.collection_one.collection"
            :values="editExisting[junctionRelatedKey]"
            @stage-value="stageValue"
          />
        </div>
      </v-modal>
    </portal>

    <portal v-if="addNew" to="modal">
      <v-modal
        :title="$t('creating_item')"
        :buttons="{
          save: {
            text: $t('save'),
            color: 'accent'
          }
        }"
        @close="addNew = false"
        @save="addNewItem"
      >
        <div class="edit-modal-body">
          <v-form
            new-item
            :fields="relation.junction.collection_one.fields"
            :collection="relation.junction.collection_one.collection"
            :values="defaultsWithEdits"
            @stage-value="stageValue"
          />
        </div>
      </v-modal>
    </portal>
  </div>
</template>

<script>
import mixin from "@directus/extension-toolkit/mixins/interface";
import shortid from "shortid";

export default {
  name: "InterfaceManyToMany",
  mixins: [mixin],
  data() {
    return {
      sort: {
        field: null,
        asc: true
      },

      selectExisting: false,
      editExisting: null,
      addNew: null,
      edits: {},

      dragging: false,

      items: [],
      loading: false,
      error: null,
      stagedSelection: null,

      stagedValue: [],
      initialValue: this.value || []
    };
  },
  computed: {
    // If the relationship has been configured or not
    relationshipSetup() {
      if (!this.relation) return false;
      return true;
    },

    // The fields that should be rendered in the modal / table
    visibleFields() {
      if (this.relationSetup === false) return [];
      if (!this.options.fields) return [];

      let visibleFieldNames;

      if (Array.isArray(this.options.fields)) {
        visibleFieldNames = this.options.fields.map(val => val.trim());
      }

      visibleFieldNames = this.options.fields.split(",").map(val => val.trim());

      // Fields in the related collection (not the JT)
      const relatedFields = this.relation.junction.collection_one.fields;

      return visibleFieldNames.map(name => {
        return relatedFields[name];
      });
    },

    visibleFieldNames() {
      return this.visibleFields.map(field => field.field);
    },

    // The name of the field that holds the primary key in the related (not JT) collection
    relatedPrimaryKeyField() {
      return _.find(this.relation.junction.collection_one.fields, { primary_key: true }).field;
    },

    selectionPrimaryKeys() {
      return (
        (this.items || [])
          // Make sure we don't try to read movies that don't exist for some reason
          .filter(item => item[this.junctionRelatedKey] !== null)
          .filter(item => item.$delete !== true)
          .map(item => item[this.junctionRelatedKey][this.relatedPrimaryKeyField])
          .filter(key => key)
      ); // Filter out empty items
    },

    // The items in this.items, but sorted by the values in this.sort
    itemsSorted: {
      get() {
        if (this.sort.field === "$manual") {
          return _.orderBy(
            this.items,
            item => item[this.sortField.field],
            this.sort.asc ? "asc" : "desc"
          );
        }

        return _.orderBy(
          this.items,
          item => item[this.junctionRelatedKey][this.sort.field],
          this.sort.asc ? "asc" : "desc"
        );
      },
      set(newValue) {
        let value = _.clone(newValue);

        this.items = value.map((item, index) => ({
          ...item,
          [this.sortField.field]: index + 1
        }));

        value = value.map((item, index) => {
          const junctionRow = {
            [this.sortField.field]: index + 1
          };

          const junctionPrimaryKey = item[this.junctionPrimaryKey.field] || null;
          const itemPrimaryKey = item[this.junctionRelatedKey][this.relatedPrimaryKeyField];
          const tempKey = item.$tempKey || null;

          if (junctionPrimaryKey) {
            junctionRow[this.junctionPrimaryKey.field] = junctionPrimaryKey;
          }

          if (itemPrimaryKey) {
            junctionRow[this.junctionRelatedKey] = {
              [this.relatedPrimaryKeyField]: itemPrimaryKey
            };
          } else {
            junctionRow[this.junctionRelatedKey] = item[this.junctionRelatedKey];
          }

          // Find and merge staged value of current row, so we don't lose made edits
          const currentRowInStaged = _.find(this.stagedValue, item => {
            return (
              item[this.junctionPrimaryKey.field] === junctionPrimaryKey ||
              item.$tempKey === tempKey
            );
          });

          const madeEdits = currentRowInStaged && currentRowInStaged[this.junctionRelatedKey];

          if (madeEdits) {
            junctionRow[this.junctionRelatedKey] = madeEdits;
          }

          return junctionRow;
        });

        this.emitValue(value);
      }
    },

    // The default values for the form fields including the edits that have been made
    defaultsWithEdits() {
      const relatedCollectionFields = this.relation.junction.collection_one.fields;

      const defaults = _.mapValues(relatedCollectionFields, field => field.default_value);

      return _.merge({}, defaults, this.edits);
    },

    // Field in the junction table that holds the sort value in the junction table
    sortField() {
      const junctionTableFields = this.relation.collection_many.fields;
      const sortField = _.find(junctionTableFields, { type: "sort" });
      return sortField;
    },

    // If the items can be manually sorted
    sortable() {
      return !!this.sortField;
    },

    manualSortActive() {
      return this.sort.field === "$manual";
    },

    // The key in the junction row that holds the data of the related item
    junctionRelatedKey() {
      return this.relation.junction.field_many.field;
    },

    junctionPrimaryKey() {
      return _.find(this.relation.junction.collection_many.fields, { primary_key: true });
    }
  },
  created() {
    if (this.sortable) {
      this.sort.field = "$manual";
    } else {
      // Set the default sort column
      this.sort.field = this.visibleFields[0].field;
    }

    // Set the initial set of items. Filter out any broken junction records
    this.items = (this.value || []).filter(item => item[this.junctionRelatedKey]);
  },
  methods: {
    // Change the sort position to the provided field. If the same field is
    // changed, flip the sort order
    changeSort(fieldName) {
      if (this.sort.field === fieldName) {
        this.sort.asc = !this.sort.asc;
        return;
      }

      this.sort.asc = true;
      this.sort.field = fieldName;
      return;
    },

    addNewItem() {
      const junctionFieldName = this.relation.junction.field_many.field;

      const newJunctionRow = {
        [junctionFieldName]: this.edits,
        $tempKey: shortid.generate()
      };

      this.items = [..._.clone(this.items), newJunctionRow];

      this.emitValue([...this.stagedValue, _.clone(newJunctionRow)]);

      this.edits = {};
      this.addNew = false;
    },

    // Save the made edits in the add new item modal
    stageValue({ field, value }) {
      this.$set(this.edits, field, value);
    },

    emitValue(newValue) {
      // Filter out the temp keys
      newValue = newValue.map(item => {
        if (item.hasOwnProperty("$tempKey")) {
          delete item.$tempKey;
        }

        return item;
      });

      this.stagedValue = newValue;
      this.$emit("input", newValue);
    },

    toggleManualSort() {
      this.sort.field = "$manual";
      this.sort.asc = true;
    },

    startEdit(primaryKey, tempKey) {
      if (!primaryKey) {
        const values = _.find(this.items, { $tempKey: tempKey });
        this.editExisting = values;
        return;
      }

      // If we're not editing a newly created item, fetch the values "fresh"
      const collection = this.relation.collection_many.collection;

      this.$api
        .getItem(collection, primaryKey, { fields: "*.*.*" })
        .then(res => res.data)
        .then(item => (this.editExisting = item))
        .catch(console.error);
    },

    saveEdits() {
      const edits = _.clone(this.edits);
      const junctionPrimaryKey = this.editExisting[this.junctionPrimaryKey.field] || null;
      const itemPrimaryKey =
        this.editExisting[this.junctionRelatedKey][this.relatedPrimaryKeyField] || null;
      const tempKey = this.editExisting.$tempKey || null;

      const junctionRow = {
        [this.junctionRelatedKey]: edits
      };

      if (junctionPrimaryKey) {
        junctionRow[this.junctionPrimaryKey.field] = junctionPrimaryKey;
      } else {
        junctionRow.$tempKey = tempKey;
      }

      if (itemPrimaryKey) {
        junctionRow[this.junctionRelatedKey][this.relatedPrimaryKeyField] = itemPrimaryKey;
      }

      let hasBeenEditedBefore = false;
      this.stagedValue.forEach(item => {
        if (
          item[this.junctionPrimaryKey.field] === junctionPrimaryKey ||
          item.$tempKey === tempKey
        ) {
          hasBeenEditedBefore = true;
        }
      });

      if (hasBeenEditedBefore) {
        this.emitValue(
          this.stagedValue.map(item => {
            if (
              item[this.junctionPrimaryKey.field] === junctionPrimaryKey ||
              item.$tempKey === tempKey
            ) {
              return junctionRow;
            }

            return item;
          })
        );
      } else {
        this.emitValue([...this.stagedValue, junctionRow]);
      }

      this.items = this.items.map(item => {
        if (
          item[this.junctionPrimaryKey.field] === junctionPrimaryKey ||
          item.$tempKey === tempKey
        ) {
          return _.merge(_.clone(item), junctionRow);
        }

        return item;
      });

      this.editExisting = null;
      this.edits = {};
    },

    stageSelection(primaryKeys) {
      this.stagedSelection = primaryKeys;
    },

    async closeSelection() {
      const primaryKeysThatAreSelected = _.clone(this.stagedSelection);
      const savedValue = _.clone(this.initialValue);

      const deletedItems = savedValue
        // Extract the items that were deleted by deselecting compared to the moment the interface
        // was loaded
        .filter(savedItem => {
          const relatedPrimaryKey = savedItem[this.junctionRelatedKey][this.relatedPrimaryKeyField];
          return primaryKeysThatAreSelected.includes(relatedPrimaryKey) === false;
        })
        // Convert the junction rows to { [id-field]: [id], $delete: true } so they will be deleted
        // by the API on save
        .map(deletedItem => {
          return {
            [this.junctionPrimaryKey.field]: deletedItem[this.junctionPrimaryKey.field],
            $delete: true
          };
        });

      const currentlySavedPrimaryKeys = savedValue.map(
        junctionRow => junctionRow[this.junctionRelatedKey][this.relatedPrimaryKeyField]
      );

      const newlyAddedItems = primaryKeysThatAreSelected
        // Extract the items that weren't selected before
        .filter(primaryKey => {
          return currentlySavedPrimaryKeys.includes(primaryKey) === false;
        })
        // Convert the newly selected items to junction row objects
        .map(primaryKey => {
          return {
            $tempKey: shortid.generate(),
            [this.junctionRelatedKey]: {
              [this.relatedPrimaryKeyField]: primaryKey
            }
          };
        });

      this.emitValue([...this.stagedValue, ...deletedItems, ...newlyAddedItems]);

      const deleteJunctionRowKeys = deletedItems.map(junctionRow => {
        return junctionRow[this.junctionPrimaryKey.field];
      });

      let newLocalItemState = savedValue
        // filter out the deleted items
        .filter(junctionRow => {
          const primaryKey = junctionRow[this.junctionPrimaryKey.field];
          return deleteJunctionRowKeys.includes(primaryKey) === false;
        })
        // Apply the previously made edits
        .map(junctionRow => {
          const primaryKey = junctionRow[this.junctionPrimaryKey.field];

          const edits = _.find(this.stagedValue, { [this.junctionPrimaryKey.field]: primaryKey });

          if (edits) {
            return _.merge({}, junctionRow, edits);
          }

          return junctionRow;
        });

      const newlyAddedItemPrimaryKeys = newlyAddedItems.map(junctionRow => {
        return junctionRow[this.junctionRelatedKey][this.relatedPrimaryKeyField];
      });

      let items = [];

      if (newlyAddedItemPrimaryKeys.length > 0) {
        // Fetch the data for the selected items fresh so we can display it on the table
        const { data } = await this.$api.getItem(
          this.relation.junction.collection_one.collection,
          newlyAddedItemPrimaryKeys,
          {
            limit: -1
          }
        );

        items = Array.isArray(data) ? data : [data];
      }

      // Augment the newly selected items with the data from the database
      const newlyAddedItemsWithData = newlyAddedItems.map(junctionRow => {
        const itemKey = junctionRow[this.junctionRelatedKey][this.relatedPrimaryKeyField];

        return _.merge({}, junctionRow, {
          [this.junctionRelatedKey]: _.find(items, { [this.relatedPrimaryKeyField]: itemKey })
        });
      });

      newLocalItemState = [...newLocalItemState, ...newlyAddedItemsWithData];
      this.items = newLocalItemState;

      this.stagedSelection = null;
      this.selectExisting = null;

      // Use initial value to create list of delete flags, only add additions for newly selected items
      // Apply changes in currentStagedValue on top of this list
      // Emit those changes

      // We need to add the $delete flag to all the items that were selected before, but aren't
      // anymore now.

      // All items that have been selected that weren't selected before need to be added

      // All edits to items that were made before need to be maintained

      // The sort order needs to be maintained
    },

    cancelSelection() {
      this.stagedSelection = null;
      this.selectExisting = null;
    }
  }
};
</script>

<style lang="scss" scoped>
.table {
  background-color: var(--white);
  border: var(--input-border-width) solid var(--lighter-gray);
  border-radius: var(--border-radius);
  border-spacing: 0;
  width: 100%;
  margin: 16px 0 24px;

  .header {
    height: var(--input-height);
    border-bottom: 2px solid var(--lightest-gray);

    button {
      text-align: left;
      font-weight: 500;
      transition: color var(--fast) var(--transition);

      &:hover {
        transition: none;
        color: var(--darker-gray);
      }
    }

    i {
      vertical-align: top;
      color: var(--light-gray);
    }
  }

  .row {
    display: flex;
    align-items: center;
    padding: 0 5px;

    > div {
      padding: 3px 5px;
      flex-basis: 200px;
    }
  }

  .header .row {
    align-items: center;
    height: 40px;

    & > button {
      padding: 3px 5px 2px;
      flex-basis: 200px;
    }
  }

  .body {
    max-height: 275px;
    overflow-y: scroll;
    -webkit-overflow-scrolling: touch;

    .row {
      cursor: pointer;
      position: relative;
      height: 50px;
      border-bottom: 2px solid var(--off-white);

      &:hover {
        background-color: var(--highlight);
      }

      & div:last-of-type {
        flex-grow: 1;
      }

      button {
        color: var(--lighter-gray);
        transition: color var(--fast) var(--transition);

        &:hover {
          transition: none;
          color: var(--danger);
        }
      }
    }
  }

  .sort-column {
    flex-basis: 36px !important;

    &.disabled i {
      color: var(--lightest-gray);
      cursor: not-allowed;
    }
  }
}

.drag-handle {
  cursor: grab;
}

.dragging {
  cursor: grabbing !important;
}

.buttons {
  margin-top: 24px;
}

.buttons > * {
  display: inline-block;
}

.buttons > *:first-child {
  margin-right: 24px;
}

.edit-modal-body {
  padding: 30px 30px 60px 30px;
  background-color: var(--body-background);
  .form {
    grid-template-columns:
      [start] minmax(0, var(--column-width)) [half] minmax(0, var(--column-width))
      [full];
  }
}
</style>
