<template>
  <div class="interface-many-to-many">
    <v-notice v-if="relationshipSetup === false" color="warning" icon="warning">
      {{ $t("relationship_not_setup") }}
    </v-notice>

    <template v-else>
      <div v-if="items.length" class="table">
        <div class="header">
          <div class="row">
            <div v-for="field in visibleFields" :key="field.field">
              {{ $helpers.formatTitle(field.field) }}
            </div>
          </div>
        </div>
        <div class="body">
          <div v-for="item in items" :key="item[relatedPrimaryKeyField]" class="row">
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
    </template>
  </div>
</template>

<script>
import mixin from "@directus/extension-toolkit/mixins/interface";

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
      stagedSelection: null
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
    }
  },
  created() {
    this.initValue();
  },
  methods: {
    // Flatten the array of junction table rows into the items in the related
    // collection
    initValue() {
      if (this.value) {
        this.items = this.value.map(junctionRow => {
          return junctionRow[this.relation.junction.field_many.field];
        });
      }
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
</style>
