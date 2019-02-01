<style lang="sass">
    .autocomplete {
        position: relative;

        input {
            width: 100%;
        }
    }

    .autocomplete__suggestions {
        position: absolute;
        top: 100%;
        background-color: #fff;
        width: 100%;
        z-index: 2;
    }

    .autocomplete__entry {
        &:hover {
            background-color: #f7f7f9;
            cursor: default;
        }
    }

    .autocomplete__selected {
        background-color: darken(#f7f7f9, 5%);
    }
</style>

<template>
    <div :class="classPrefix" @mousedown="mousefocus = true" @mouseout="mousefocus = false">
        <input type="text" @blur="focused = false" @focus="focused = true"
            v-model="search" :class="inputClass"
            @keydown.down.prevent.stop="moveDown()"
            @keydown.up.prevent.stop="moveUp()"
            @keydown.enter.prevent="select(selectedIndex, $event)"
            @keydown.tab="mousefocus = false"
            v-bind="$attrs"
            ref="input">
        <div v-if="showSuggestions" :class="classPrefix + '__suggestions'">
            <div v-for="(entry, index) in filteredEntries"
                @click="select(index, $event)"
                :class="[classPrefix + '__entry', selectedClass(index)]">
                {{ getProperty(entry) }}
          </div>
        </div>
    </div>
</template>
<script>
import { autocompleteBus } from "./autocompleteBus.js";

export default {
  inheritAttrs: false,
  data() {
    return {
      entries: [],
      search: this.value || "",
      focused: false,
      mousefocus: false,
      selectedIndex: -1
    };
  },
  computed: {
    filteredEntries() {
      if (this.search.length <= this.threshold) {
        return [];
      }
      var search = this.ignoreCase ? this.search.toLowerCase() : this.search;
      return this.entries.filter(entry => {
        var current = this.getProperty(entry);
        if (this.ignoreCase) {
          current = current.toLowerCase();
        }
        return current.indexOf(search) > -1;
      });
    },
    hasSuggestions() {
      if (this.search.length <= this.threshold) {
        return false;
      }

      return this.filteredEntries.length > 0;
    },
    showSuggestions() {
      if (!this.hasSuggestions) {
        return false;
      }

      if (this.focused || this.mousefocus) {
        return true;
      }

      return false;
    }
  },
  created() {
    this.search = this.value;
    if (this.list !== undefined) {
      this.setEntries(this.list);
    } else if (this.url !== undefined && this.requestType !== undefined) {
      this.getListAjax();
    }
  },
  methods: {
    getProperty(entry) {
      if (typeof (this.property) == "function") {
        return this.property(entry);
      } else if (typeof (this.property) == "string") {
        return entry[this.property];
      } else {
        throw "Unsupported type";
      }
    },
    select(index, event) {
      if (!this.hasSuggestions) return;
      if (index === -1) return;

      event.stopPropagation();
      this.search = this.getProperty(this.filteredEntries[index]);
      autocompleteBus.$emit("autocomplete-select", this.search);
      this.$emit("selected", this.filteredEntries[index]);

      if (this.autoHide) {
        this.mousefocus = false;
        this.focused = false;
        this.$refs.input.blur();
      } else {
        this.$nextTick(() => {
          this.$refs.input.focus();
        });
      }
    },
    setEntries(list) {
      this.entries = list;
    },
    moveUp() {
      if (this.selectedIndex - 1 < -1) {
        this.selectedIndex = this.filteredEntries.length - 1;
      } else {
        this.selectedIndex -= 1;
      }
    },
    moveDown() {
      if (this.selectedIndex + 1 > this.filteredEntries.length - 1) {
        this.selectedIndex = -1;
      } else {
        this.selectedIndex += 1;
      }
    },
    selectedClass(index) {
      if (index === this.selectedIndex) {
        return this.classPrefix + "__selected";
      }

      return "";
    },
    getListAjax() {
      return this.$http[this.requestType](this.url).then(response => {
        this.setEntries(response.data);
      });
    }
  },
  props: {
    classPrefix: {
      type: String,
      required: false,
      default: "autocomplete"
    },
    url: {
      type: String,
      required: false
    },
    requestType: {
      type: String,
      required: false,
      default: "get"
    },
    list: {
      type: Array,
      required: false
    },
    property: {
      type: [String, Function],
      required: false,
      default: "name"
    },
    inputClass: {
      type: String,
      required: false
    },
    ignoreCase: {
      type: Boolean,
      required: false,
      default: true
    },
    threshold: {
      type: Number,
      required: false,
      default: 0
    },
    value: {
      required: false,
      default: ""
    },
    autoHide: {
      type: Boolean,
      required: false,
      default: true
    }
  },
  watch: {
    filteredEntries(value) {
      if (this.selectedIndex > value.length - 1) {
        this.selectedIndex = 0;
      }
    },
    search(value) {
      this.$emit("input", value);
    },
    value(newValue) {
      this.search = newValue;
    },
    list(value) {
      this.setEntries(value);
    }
  }
};
</script>
