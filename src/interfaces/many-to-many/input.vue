<template>
  <div class="interface-many-to-many">
    <v-notice v-if="relationshipSetup === false" color="warning" icon="warning">
      {{ $t("relationship_not_setup") }}
    </v-notice>

    <template v-else>
      <div v-if="items.length" class="table">
        <div class="header">
          <div class="row">
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
        <div class="body">
          <div v-for="item in itemsSorted" :key="item[relatedPrimaryKeyField]" class="row">
            <v-ext-display
              v-for="field in visibleFields"
              :key="field.field"
              :interface-type="field.interface"
              :name="field.field"
              :type="field.type"
              :collection="field.collection"
              :datatype="field.datatype"
              :options="field.options"
              :value="item[field.field]"
            />
          </div>
        </div>
      </div>

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
      </div>
    </template>

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

      initialValue: this.value,

      stagedValue: []
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

    // The name of the field that holds the primary key in the related (not JT)
    // collection
    relatedPrimaryKeyField() {
      return _.find(this.relation.junction.collection_one.fields, { primary_key: true }).field;
    },

    // The items in this.items, but sorted by the values in this.sort
    itemsSorted() {
      return _.orderBy(this.items, item => item[this.sort.field], this.sort.asc ? "asc" : "desc");
    },

    defaultsWithEdits() {
      const relatedCollectionFields = this.relation.junction.collection_one.fields;

      const defaults = _.mapValues(relatedCollectionFields, field => field.default_value);

      return _.merge({}, defaults, this.edits);
    }
  },
  created() {
    // Flatten the array of junction table rows into the items in the related
    // collection. This means we keep a different local state vs the changes
    // we stage.
    if (this.value && this.value.length > 0) {
      this.items = this.value
        .map(junctionRow => {
          return junctionRow[this.relation.junction.field_many.field];
        })
        // Filter out junction row items where the related movie is null
        .filter(item => item);
    }

    // Set the default sort column
    this.sort.field = this.visibleFields[0].field;
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
      const value = _.clone(this.defaultsWithEdits);

      const newJunctionRow = {
        [junctionFieldName]: this.edits
      };

      value.$tempKey = shortid.generate();

      this.emitValue([...this.stagedValue, newJunctionRow]);

      this.items = [...this.items, value];

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
