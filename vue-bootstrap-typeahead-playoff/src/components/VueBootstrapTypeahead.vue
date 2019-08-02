<template>
  <div>
    <div :class="sizeClasses">
      <div ref="prependDiv" v-if="$slots.prepend || prepend" class="input-group-prepend">
        <slot name="prepend">
          <span class="input-group-text">{{ prepend }}</span>
        </slot>
      </div>
      <input
        ref="input"
        type="search"
        :class="`form-control ${inputClass}`"
        :placeholder="placeholder"
        :aria-label="placeholder"
        :value="inputValue"
        @focus="isFocused = true"
        @blur="handleBlur"
        @input="handleInput($event.target.value)"
        @keydown="moveArrows($event)"
        autocomplete="off"
        :name="inputName"
        :id="inputId"
      />
      <div v-if="$slots.append || append" class="input-group-append">
        <slot name="append">
          <span class="input-group-text">{{ append }}</span>
        </slot>
      </div>
    </div>
    <vue-bootstrap-typeahead-list
      class="vbt-autcomplete-list"
      ref="list"
      :style="{width: width}"
      v-show="isFocused"
      :query="inputValue"
      :data="formattedData"
      :background-variant="backgroundVariant"
      :text-variant="textVariant"
      :maxMatches="maxMatches"
      :minMatchingChars="minMatchingChars"
      :notFoundText="notFoundText"
      :showNotFound="showNotFound"
      @hit="handleHit"
    >
      <!-- pass down all scoped slots -->
      <template v-for="(slot, slotName) in $scopedSlots" :slot="slotName" slot-scope="{ data, htmlText }">
        <slot :name="slotName" v-bind="{ data, htmlText }"></slot>
      </template>
    </vue-bootstrap-typeahead-list>
  </div>
</template>

<script>
import VueBootstrapTypeaheadList from './VueBootstrapTypeaheadList.vue'
import ResizeObserver from 'resize-observer-polyfill'

export default {
  name: 'VueBootstrapTypehead',

  components: {
    VueBootstrapTypeaheadList
  },

  props: {
    size: {
      type: String,
      default: null,
      validator: size => ['lg', 'sm'].indexOf(size) > -1
    },
    value: {
      type: String
    },
    data: {
      type: Array,
      required: true,
      validator: d => d instanceof Array
    },
    serializer: {
      type: Function,
      default: (d) => d,
      validator: d => d instanceof Function
    },
    backgroundVariant: String,
    textVariant: String,
    inputClass: {
      type: String,
      default: ''
    },
    maxMatches: {
      type: Number,
      default: 10
    },
    minMatchingChars: {
      type: Number,
      default: 2
    },
    placeholder: String,
    prepend: String,
    append: String,
    notFoundText: {
      type: String,
      default: ""
    },
    showNotFound: {
      type: Boolean,
      default: false
    },
    width: {
      type: String,
      default: "auto"
    },
    inputName: {
      type: String,
      default: ""
    },
    inputId: {
      type: String,
      default: ""
    }
  },

  computed: {
    sizeClasses() {
      return this.size ? `input-group input-group-${this.size}` : 'input-group'
    },

    formattedData() {
      if (!(this.data instanceof Array)) {
        return []
      }
      return this.data.map((d, i) => {
        return {
          id: i,
          data: d,
          text: this.serializer(d),
          selected: this.indexArrowItem == i
        }
      })
    }
  },

  methods: {
    resizeList(el) {
      const rect = el.getBoundingClientRect()
      const listStyle = this.$refs.list.$el.style
      // Set the width of the list on resize
      rect.width = this.width == "auto" ? rect.width : this.width;
      listStyle.width = rect.width + 'px'
      // Set the margin when the prepend prop or slot is populated
      // (setting the "left" CSS property doesn't work)
      if (this.$refs.prependDiv) {
        const prependRect = this.$refs.prependDiv.getBoundingClientRect()
        listStyle.marginLeft = prependRect.width + 'px'
      }
    },

    handleHit(evt) {
      if (typeof this.value !== 'undefined') {
        this.$emit('input', evt.text)
      }
      
      this.inputValue = evt.text
      this.$emit('hit', evt.data)
      this.$refs.input.blur()
      this.isFocused = false
    },

    handleBlur(evt) {
      const tgt = evt.relatedTarget
      if (tgt && tgt.classList.contains('vbst-item')) {
        return
      }
      this.isFocused = false
    },

    handleInput(newValue) {
      this.inputValue = newValue

      // If v-model is being used, emit an input event
      if (typeof this.value !== 'undefined') {
        this.$emit('input', newValue)
      }
    },

    moveArrows(event) {
      if(!this.isFocused) {
        return false;
      }

      switch(event.code) {
        case "ArrowDown":
          event.preventDefault();
          if(this.indexArrowItem == null) {
            this.indexArrowItem = 0;
          } else if(this.indexArrowItem != null && this.indexArrowItem < this.data.length - 1) {
            this.indexArrowItem++;
          }
          this.$forceUpdate();

          break;
        case "ArrowUp":
          event.preventDefault();
          if(this.indexArrowItem == null) {
            this.indexArrowItem = 0;
          } else if(this.indexArrowItem != null && this.indexArrowItem > 0) {
            this.indexArrowItem--;
          }
          this.$forceUpdate();

          break;
        case "Enter":
          event.preventDefault();
          var selectedItem = this.formattedData.filter((item, index) => {
            return item.selected;
          });

          if(selectedItem.length == 0) {
            return false;
          }

          this.handleHit(selectedItem[0]);
          break;
      }
    }
  },

  data() {
    return {
      indexArrowItem: null,
      isFocused: false,
      inputValue: ''
    }
  },

  mounted() {
    this.$_ro = new ResizeObserver(e => {
      this.resizeList(this.$refs.input)
    })
    this.$_ro.observe(this.$refs.input)
    this.$_ro.observe(this.$refs.list.$el)
  },

  beforeDestroy() {
    this.$_ro.disconnect()
  }
}
</script>

<style scoped>
  .vbt-autcomplete-list {
    padding-top: 5px;
    position: absolute;
    max-height: none;
    overflow-y: auto;
    z-index: 999;
  }
</style>
