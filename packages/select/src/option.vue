<template>
  <li
    @mouseenter="hoverItem"
    @click.stop="selectOptionClick"
    class="el-select-dropdown__item"
    v-show="visible"
    :class="{
      'selected': itemSelected,
      'is-disabled': disabled || groupDisabled || limitReached,
      'hover': hover
    }">
    <slot>
      <span>{{ currentLabel }}</span>
    </slot>
  </li>
</template>

<script type="text/babel">
  import Emitter from 'element-ui/src/mixins/emitter';
  import { getValueByPath, escapeRegexpString } from 'element-ui/src/utils/util';

  export default {
    mixins: [Emitter],

    name: 'ElOption',

    componentName: 'ElOption',

    inject: ['select'],

    props: {
      value: {
        required: true
      },
      label: [String, Number],
      created: Boolean,
      disabled: {
        type: Boolean,
        default: false
      }
    },

    data() {
      return {
        index: -1,
        groupDisabled: false,
        visible: true,
        hitState: false,
        hover: false
      };
    },

    computed: {
      isObject() {
        return Object.prototype.toString.call(this.value).toLowerCase() === '[object object]';
      },

      currentLabel() {
        return this.label || (this.isObject ? '' : this.value); // 如果传入对象，当做空字符串展示
      },

      currentValue() {
        return this.value || this.label || ''; // 没有传value，用label展示，所以调用时，若value和label一致，传一个即可。
      },
      // 检测是否可选
      itemSelected() {
        if (!this.select.multiple) {
          return this.isEqual(this.value, this.select.value);
        } else {
          return this.contains(this.select.value, this.value); // 新写法：计算属性里return函数，函数里就可以操作数据
        }
      },
      // 检测是否达到可选上限
      limitReached() {
        if (this.select.multiple) {
          return !this.itemSelected &&
            (this.select.value || []).length >= this.select.multipleLimit &&
            this.select.multipleLimit > 0;
        } else {
          return false;
        }
      }
    },

    watch: {
      currentLabel() {
        if (!this.created && !this.select.remote) this.dispatch('ElSelect', 'setSelected');
      },
      value(val, oldVal) {
        const { remote, valueKey } = this.select;
        if (!this.created && !remote) {
          if (valueKey && typeof val === 'object' && typeof oldVal === 'object' && val[valueKey] === oldVal[valueKey]) {
            return;
          }
          this.dispatch('ElSelect', 'setSelected');
        }
      }
    },

    methods: {
      // 单选判断是否已选
      isEqual(a, b) {
        if (!this.isObject) {
          return a === b;
        } else {
          const valueKey = this.select.valueKey;
          return getValueByPath(a, valueKey) === getValueByPath(b, valueKey);
        }
      },
      // 多选判断是否已选
      contains(arr = [], target) {
        if (!this.isObject) {
          return arr && arr.indexOf(target) > -1;
        } else {
          const valueKey = this.select.valueKey;
          return arr && arr.some(item => {
            return getValueByPath(item, valueKey) === getValueByPath(target, valueKey);
          });
        }
      },

      handleGroupDisabled(val) {
        this.groupDisabled = val;
      },

      hoverItem() {
        // 鼠标hover后，执行修改父组件里的hoverIndex值
        if (!this.disabled && !this.groupDisabled) {
          this.select.hoverIndex = this.select.options.indexOf(this);
        }
      },

      selectOptionClick() {
        // @click事件选中option，告诉爸爸选中[当前vue、true为选中]
        if (this.disabled !== true && this.groupDisabled !== true) {
          this.dispatch('ElSelect', 'handleOptionClick', [this, true]);
        }
      },

      // 父元素el-select会触发该事件，做了$on的监听。
      queryChange(query) {
        let { isCheckedAll, filteredOptions, filteredOptionsVal, filteredDisabledList, cachedOptions} = this.select;
        // 如果是需要的内容或者是创建的，就展示
        this.visible = new RegExp(escapeRegexpString(query), 'i').test(this.currentLabel) || this.created;
        if (!this.visible) {
          // 如果当前不展示，过滤列表数量-1
          this.select.filteredOptionsCount--;
          if (isCheckedAll) {
            let val = this.currentValue + this.currentLabel;
            // 过滤列表里 -1
            let index = filteredOptions.indexOf(this);
            this.select.filteredOptions.splice(index, 1);
            // 整理目前列表数据，由value+label的快捷内容组成，方便查询该项是否存在
            let indexVal = filteredOptionsVal.indexOf(val);
            this.select.filteredOptionsVal.splice(indexVal, 1);
            // 整理过滤后列表内排除disabled后的内容，由value+label的快捷内容组成，方便使用其length管理全选的状态
            if (!this.disabled) {
              let disableIndex = filteredDisabledList.indexOf(val);
              this.select.filteredDisabledList.splice(disableIndex, 1);
            }
          }
        } else {
          // 过滤列表里+1
          if (isCheckedAll) {
            // 如果过滤列表是满的，不再增加
            if (filteredOptions.length >= cachedOptions.length) return;
            // 如果过滤列表不满、且列表内不存在当前项，增加
            let val = this.currentValue + this.currentLabel;
            if (!filteredOptionsVal.includes(val)) {
              this.select.filteredOptions.push(this); // 过滤列表缓存每一项
              this.select.filteredOptionsVal.push(val);
              if (!this.disabled) {
                this.select.filteredDisabledList.push(val);
              }
            }
          }
        }
      }
    },

    created() {
      let val = this.currentValue + this.currentLabel;
      this.select.options.push(this); // options里为每一项的vue对象，
      this.select.cachedOptions.push(this); // 缓存每一项
      if (this.select.isCheckedAll) {
        this.select.filteredOptions.push(this); // 过滤列表缓存每一项
        this.select.filteredOptionsVal.push(val);
      }
      if (!this.disabled) {
        // 缓存没有disabled的过滤列表
        this.select.cachedOptionsDisable.push(this);
        this.select.filteredDisabledList.push(val);
      }
      this.select.optionsCount++;
      this.select.filteredOptionsCount++;
      this.$on('queryChange', this.queryChange); // $on的写法还可以这样，不用绑定在组件标签上？
      this.$on('handleGroupDisabled', this.handleGroupDisabled);
    },

    beforeDestroy() {
      const { selected, multiple } = this.select;
      let selectedOptions = multiple ? selected : [selected];
      let index = this.select.cachedOptions.indexOf(this);
      let selectedIndex = selectedOptions.indexOf(this);

      // if option is not selected, remove it from cache
      if (index > -1 && selectedIndex < 0) {
        this.select.cachedOptions.splice(index, 1);
      }
      this.select.onOptionDestroy(this.select.options.indexOf(this));
    }
  };
</script>
