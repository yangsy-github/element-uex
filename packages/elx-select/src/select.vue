<template>
  <div
    class="el-select"
    v-clickoutside="handleClose"
    :class="{ 'is-multiple': multiple, 'is-small': size === 'small'}">
    <div class="el-select__tags" v-if="multiple" @click.stop="toggleMenu" ref="tags" :style="{ 'max-width': inputWidth - 32 + 'px' }">
      <transition-group @after-leave="resetInputHeight">
        <el-tag
          v-for="item in selected"
          :key="item.value"
          closable
          :hit="item.hitState"
          type="primary"
          @close="deleteTag($event, item)"
          close-transition>{{ item.currentLabel }}</el-tag>
      </transition-group>
    </div>
    <el-input
      ref="reference"
      v-model="selectedLabel"
      type="text"
      :placeholder="currentPlaceholder"
      :name="name"
      :disabled="disabled"
      readonly="readonly"
      @focus="toggleMenu"
      @click="toggleMenu"
      @mousedown.native="handleMouseDown"
      @mouseenter.native="inputHovering = true"
      @mouseleave.native="inputHovering = false"
      :icon="iconClass">
    </el-input>
    <transition name="md-fade-bottom" @after-leave="doDestroy">
      <el-select-menu
        ref="popper"
        v-show="visible"
        :class="{'card': type === 'card'}">
        <el-input
            v-if='filterable||(multiple&&filterable)'
            class="el-select-dropdown__search"
            :placeholder="t('el.select.inputKw')"
            v-model="query"
            size="small"
            :debounce="remote ? 300 : 0"
            @click="toggleMenu"
            @keyup.native="debouncedOnInputChange"
            @keydown.native.down.prevent="navigateOptions('next')"
            @keydown.native.up.prevent="navigateOptions('prev')"
            @keydown.native.enter.prevent="selectOption"
            @keydown.native.esc.prevent="visible = false"
            @keydown.native.tab="visible = false">
        </el-input>
        <ul class="el-select-dropdown__list" v-show="options.length > 0 && filteredOptionsCount > 0 && !loading">
          <slot></slot>
        </ul>
        <p class="el-select-dropdown__empty" v-if="emptyText">{{ emptyText }}</p>
      </el-select-menu>
    </transition>
  </div>
</template>

<script type="text/babel">
  import Emitter from 'element-uex/src/mixins/emitter';
  import Locale from 'element-uex/src/mixins/locale';
  import debounce from 'throttle-debounce/debounce';
  import ElSelectMenu from './select-dropdown.vue';
  import ElOption from './option.vue';
  import Clickoutside from 'element-uex/src/utils/clickoutside';
  import { addClass, removeClass, hasClass } from 'wind-dom/src/class';
  import { addResizeListener, removeResizeListener } from 'element-uex/src/utils/resize-event';
  import { t } from 'element-uex/src/locale';

  export default {
    mixins: [Emitter, Locale],

    name: 'elx-select',

    componentName: 'elx-select',

    computed: {
      iconClass() {
        return this.showCloseIcon ? 'circle-close' : (this.remote && this.filterable ? 'caret-top' : 'caret-top');
      },

      debounce() {
        return this.remote ? 300 : 0;
      },

      showCloseIcon() {
        let criteria = this.clearable && this.inputHovering && !this.multiple && this.options.indexOf(this.selected) > -1;
        if (!this.$el) return false;

        let icon = this.$el.querySelector('.el-input__icon');
        if (icon) {
          if (criteria) {
            addClass(icon, 'is-show-close');
          } else {
            removeClass(icon, 'is-show-close');
          }
        }
        return criteria;
      },

      emptyText() {
        if (this.loading) {
          return this.t('el.select.loading');
        } else {
          if (this.voidRemoteQuery) {
            this.voidRemoteQuery = false;
            return false;
          }
          if (this.filterable && this.filteredOptionsCount === 0) {
            return this.t('el.select.noMatch');
          }
          if (this.options.length === 0) {
            return this.t('el.select.noData');
          }
        }
        return null;
      }
    },

    directives: { Clickoutside },

    components: {
      ElSelectMenu,
      ElOption
    },

    props: {
      root: Object,
      name: String,
      type: String,
      value: {},
      size: String,
      isPostRoot: Boolean,
      disabled: Boolean,
      clearable: Boolean,
      filterable: Boolean,
      loading: Boolean,
      remote: Boolean,
      remoteMethod: Function,
      filterMethod: Function,
      multiple: Boolean,
      placeholder: {
        type: String,
        default() {
          return t('el.select.placeholder');
        }
      }
    },

    data() {
      return {
        options: [],
        selected: {},
        isSelect: true,
        inputLength: 20,
        inputWidth: 0,
        valueChangeBySelected: false,
        cachedPlaceHolder: '',
        optionsCount: 0,
        filteredOptionsCount: 0,
        dropdownUl: null,
        visible: false,
        selectedLabel: '',
        query: '',
        selectInit: false,
        hoverIndex: -1,
        voidRemoteQuery: false,
        bottomOverflowBeforeHidden: 0,
        optionsAllDisabled: false,
        inputHovering: false,
        currentPlaceholder: ''
      };
    },

    watch: {
      placeholder(val) {
        this.currentPlaceholder = val;
      },

      value(val, oldVal) {
        this.changeValue();
        if (Array.isArray(val)) {
          if (JSON.stringify(val) !== JSON.stringify(oldVal)) {
            this.$emit('change', val);
            this.dispatch('ElFormItem', 'el.form.change', val);
          } else {
            return;
          }
        }
        this.$emit('change', val);
        this.dispatch('ElFormItem', 'el.form.change', val);
      },

      selected(val, oldVal) {
        if (this.multiple) {
          if (this.selected.length > 0) {
            this.currentPlaceholder = '';
          } else {
            this.currentPlaceholder = this.cachedPlaceHolder;
          }
          this.$nextTick(() => {
            this.resetInputHeight();
          });
          if (this.selectedInit) {
            this.selectedInit = false;
            return;
          }
          this.valueChangeBySelected = true;
          const result = val.map(item => item.value);
          this.$emit('input', result);
          if (this.filterable) {

            this.hoverIndex = -1;
            this.inputLength = 20;
          }
        } else {
          if (this.selectedInit) {
            this.selectedInit = false;
            return;
          }
          if (val.value === oldVal.value) return;
          this.$emit('input', val.value);
        }
      },
      query(val) {
        this.$nextTick(() => {
          this.broadcast('select-dropdown', 'updatePopper');
        });
        if (this.multiple && this.filterable) {
          this.resetInputHeight();
        }
        if (this.remote && typeof this.remoteMethod === 'function') {
          this.hoverIndex = -1;
          if (this.isPostRoot) {
            this.remoteMethod(this.root, val);
          } else {
            this.remoteMethod(val);
          }
          this.voidRemoteQuery = val === '';
          this.broadcast('elx-option', 'resetIndex');
        } else if (typeof this.filterMethod === 'function') {
          this.filterMethod(val);
        } else {
          this.filteredOptionsCount = this.optionsCount;
          this.broadcast('elx-option', 'queryChange', val);
        }
      },
      visible(val) {
        if (!val) {
          this.$refs.reference.$el.querySelector('input').blur();
          if (this.$el.querySelector('.el-input__icon')) {
            removeClass(this.$el.querySelector('.el-input__icon'), 'is-reverse');
          }
          this.broadcast('select-dropdown', 'destroyPopper');
          if (this.$refs.input) {
            this.$refs.input.blur();
          }
          this.resetHoverIndex();
          if (!this.multiple) {
            if (this.dropdownUl && this.selected.$el) {
              this.bottomOverflowBeforeHidden = this.selected.$el.getBoundingClientRect().bottom - this.$refs.popper.$el.getBoundingClientRect().bottom;
            }
            if (this.selected && this.selected.value) {
              this.selectedLabel = this.selected.currentLabel;
            }
          }
        } else {
          let icon = this.$el.querySelector('.el-input__icon');
          if (icon && !hasClass(icon, 'el-icon-circle-close')) {
            addClass(this.$el.querySelector('.el-input__icon'), 'is-reverse');
          }
          this.broadcast('select-dropdown', 'updatePopper');
          if (this.filterable) {
            this.broadcast('input', 'inputSelect');
          }
          if (!this.dropdownUl) {
            let dropdownChildNodes = this.$refs.popper.$el.childNodes;
            this.dropdownUl = [].filter.call(dropdownChildNodes, item => item.tagName === 'UL')[0];
          }
          if (this.dropdownUl) {
            if (this.bottomOverflowBeforeHidden > 0) {
              this.dropdownUl.scrollTop += this.bottomOverflowBeforeHidden;
            }
          }
        }
        if (val) {
          this.$emit('visible-change', val);
        }
      },

      options(val) {
        this.optionsAllDisabled = val.length === val.filter(item => item.disabled === true).length;
        let inputs = this.$el.querySelectorAll('input');
        if ([].indexOf.call(inputs, document.activeElement) === -1) {
          this.changeValue();
        }
      }
    },

    methods: {
      changeValue: function() {
        var val = this.value;
        if (this.valueChangeBySelected) {
          this.valueChangeBySelected = false;
          return;
        }
        this.$nextTick(() => {
          if (this.multiple && Array.isArray(val)) {
            this.$nextTick(() => {
              this.resetInputHeight();
            });
            this.selectedInit = true;
            this.selected = [];
            this.currentPlaceholder = this.cachedPlaceHolder;
            val.forEach(item => {
              let option = this.options.filter(option => option.value === item)[0];
              if (option) {
                this.$emit('addOptionToValue', option);
              }
            });
          }
          if (!this.multiple) {
            let option = this.options.filter(option => option.value === val)[0];
            if (option) {
              this.$emit('addOptionToValue', option);
            } else {
              const label = typeof val === 'string' || typeof val === 'number' ? val : '';
              let newOption = {
                value: val,
                currentLabel: label
              };
              newOption.hitState = false;
              this.selected = newOption;
              this.selectedLabel = label;
            }
          }
          this.resetHoverIndex();
        });
      },
      handleMouseDown(event) {
        if (event.target.tagName !== 'INPUT') return;
        if (this.visible) {
          this.handleClose();
          event.preventDefault();
        }
      },

      doDestroy() {
        this.$refs.popper && this.$refs.popper.doDestroy();
      },

      handleClose() {
        this.visible = false;
      },

      toggleLastOptionHitState(hit) {
        if (!Array.isArray(this.selected)) return;
        const option = this.selected[this.selected.length - 1];
        if (!option) return;

        if (hit === true || hit === false) {
          option.hitState = hit;
          return hit;
        }

        option.hitState = !option.hitState;
        return option.hitState;
      },

      deletePrevTag(e) {
        if (e.target.value.length <= 0 && !this.toggleLastOptionHitState()) {
          this.selected.pop();
        }
      },

      addOptionToValue(option, init) {
        if (this.multiple) {
          if (this.selected.indexOf(option) === -1) {
            this.selectedInit = !!init;
            this.selected.push(option);
            this.resetHoverIndex();
          }
        } else {
          this.selectedInit = !!init;
          this.selected = option;
          this.selectedLabel = option.currentLabel;
          this.hoverIndex = option.index;
        }
      },

      resetInputHeight() {
        this.$nextTick(() => {
          let inputChildNodes = this.$refs.reference.$el.childNodes;
          let input = [].filter.call(inputChildNodes, item => item.tagName === 'INPUT')[0];
          input.style.height = Math.max(this.$refs.tags.clientHeight + 6, this.size === 'small' ? 28 : 36) + 'px';
          this.broadcast('select-dropdown', 'updatePopper');
        });
      },

      resetHoverIndex() {
        setTimeout(() => {
          if (!this.multiple) {
            this.hoverIndex = this.options.indexOf(this.selected);
          } else {
            if (this.selected.length > 0) {
              this.hoverIndex = Math.min.apply(null, this.selected.map(item => this.options.indexOf(item)));
            } else {
              this.hoverIndex = -1;
            }
          }
        }, 300);
      },

      handleOptionSelect(option) {
        if (!this.multiple) {
          this.selected = option;
          this.selectedLabel = option.currentLabel;
          this.visible = false;
        } else {
          let optionIndex = -1;
          this.selected.forEach((item, index) => {
            if (item === option || item.currentLabel === option.currentLabel) {
              optionIndex = index;
            }
          });
          if (optionIndex > -1) {
            this.selected.splice(optionIndex, 1);
          } else {
            this.selected.push(option);
          }
        }
      },

      toggleMenu(event) {
        if (event.target.tagName !== 'INPUT') {
          var tagClass = event.target.getAttribute('class');
          if (tagClass.indexOf('circle-close') > -1) {
            this.deleteSelected(event);
            return;
          }
        }
        if (this.filterable && this.query === '' && this.visible) {
          return;
        }
        if (!this.disabled) {
          this.visible = !this.visible;
          this.query = '';
        }
      },

      navigateOptions(direction) {
        if (!this.visible) {
          this.visible = true;
          return;
        }
        if (!this.optionsAllDisabled) {
          if (direction === 'next') {
            this.hoverIndex++;
            if (this.hoverIndex === this.options.length) {
              this.hoverIndex = 0;
            }
            this.resetScrollTop();
            if (this.options[this.hoverIndex].disabled === true ||
              this.options[this.hoverIndex].groupDisabled === true ||
              !this.options[this.hoverIndex].visible) {
              this.navigateOptions('next');
            }
          }
          if (direction === 'prev') {
            this.hoverIndex--;
            if (this.hoverIndex < 0) {
              this.hoverIndex = this.options.length - 1;
            }
            this.resetScrollTop();
            if (this.options[this.hoverIndex].disabled === true ||
              this.options[this.hoverIndex].groupDisabled === true ||
              !this.options[this.hoverIndex].visible) {
              this.navigateOptions('prev');
            }
          }
        }
      },

      resetScrollTop() {
        let bottomOverflowDistance = this.options[this.hoverIndex].$el.getBoundingClientRect().bottom - this.$refs.popper.$el.getBoundingClientRect().bottom;
        let topOverflowDistance = this.options[this.hoverIndex].$el.getBoundingClientRect().top - this.$refs.popper.$el.getBoundingClientRect().top;
        if (bottomOverflowDistance > 0) {
          this.dropdownUl.scrollTop += bottomOverflowDistance;
        }
        if (topOverflowDistance < 0) {
          this.dropdownUl.scrollTop += topOverflowDistance;
        }
      },

      selectOption() {
        if (this.options[this.hoverIndex]) {
          this.handleOptionSelect(this.options[this.hoverIndex]);
        }
      },

      deleteSelected(event) {
        event.stopPropagation();
        this.selected = {};
        this.selectedLabel = '';
        this.$emit('input', '');
        this.$emit('change', '');
        this.visible = false;
      },

      deleteTag(event, tag) {
        let index = this.selected.indexOf(tag);
        if (index > -1) {
          this.selected.splice(index, 1);
        }
        event.stopPropagation();
      },

      onInputChange() {
        if (this.filterable && this.selectedLabel !== this.value) {
        }
      },

      onOptionDestroy(option) {
        this.optionsCount--;
        this.filteredOptionsCount--;
        let index = this.options.indexOf(option);
        if (index > -1) {
          this.options.splice(index, 1);
        }
        this.broadcast('option', 'resetIndex');
      },

      resetInputWidth() {
        this.inputWidth = this.$refs.reference.$el.getBoundingClientRect().width;
      }
    },

    created() {
      this.cachedPlaceHolder = this.currentPlaceholder = this.placeholder;
      if (this.multiple) {
        this.selectedInit = true;
        this.selected = [];
      }
      if (this.remote) {
        this.voidRemoteQuery = true;
      }

      this.debouncedOnInputChange = debounce(this.debounce, () => {
        this.onInputChange();
      });
      this.changeValue();
      this.$on('addOptionToValue', this.addOptionToValue);
      this.$on('handleOptionClick', this.handleOptionSelect);
      this.$on('onOptionDestroy', this.onOptionDestroy);
    },

    mounted() {
      addResizeListener(this.$el, this.resetInputWidth);
      if (this.remote && this.multiple && Array.isArray(this.value)) {
        this.selected = this.options.reduce((prev, curr) => {
          return this.value.indexOf(curr.value) > -1 ? prev.concat(curr) : prev;
        }, []);
        this.$nextTick(() => {
          this.resetInputHeight();
        });
      }
      this.$nextTick(() => {
        if (this.$refs.reference.$el) {
          this.inputWidth = this.$refs.reference.$el.getBoundingClientRect().width;
        }
      });
    },

    destroyed() {
      if (this.resetInputWidth) removeResizeListener(this.$el, this.resetInputWidth);
    }
  };
</script>
