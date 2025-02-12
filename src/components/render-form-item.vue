<template>
  <el-form-item
    v-show="_show"
    :prop="prop"
    :label="data.label"
    :rules="!readonly && Array.isArray(data.rules) ? data.rules : []"
    v-bind="data.attrs"
    class="render-form-item"
  >
    <template v-if="readonly && hasReadonlyContent">
      <el-input
        v-if="data.type === 'input'"
        v-bind="componentProps"
        :value="itemValue"
        readonly
        v-on="listeners"
      />
      <div v-else-if="data.type === 'select'">
        <template>
          {{ multipleValue }}
        </template>
      </div>
    </template>
    <search-select-input
      v-else-if="data.type === 'select-input'"
      :options="options"
      :value="itemValue"
      v-on="listeners"
    />
    <custom-component
      v-else
      :component="data.component || `el-${data.type}`"
      v-bind="componentProps"
      :value="itemValue"
      :disabled="disabled || componentProps.disabled || readonly"
      v-on="listeners"
    >
      <template v-for="opt in options">
        <el-option
          v-if="data.type === 'select'"
          :key="opt.value"
          v-bind="opt"
        />
        <!-- TODO: 支持 el-checkbox-button 变体 -->
        <el-checkbox
          v-else-if="data.type === 'checkbox-group'"
          :key="opt.label"
          v-bind="opt"
        />
        <!-- WARNING: radio 用 label 属性来表示 value 的含义 -->
        <!-- FYI: radio 的 value 属性可以在没有 radio-group 时用来关联到同一个 v-model -->
        <el-radio
          v-else-if="data.type === 'radio-group'"
          :key="opt.label"
          v-bind="opt"
          :label="'value' in opt ? opt.value : opt.label"
          >{{ opt.label }}</el-radio
        >
      </template>
    </custom-component>
  </el-form-item>
</template>
<script>
import searchSelectInput from './search-select-input.vue'
import getEnableWhenStatus from '../util/enable-when'
import {noop} from '../util/utils'
import _get from 'lodash.get'
import _includes from 'lodash.includes'
import _topairs from 'lodash.topairs'
import _frompairs from 'lodash.frompairs'

function validator(data) {
  if (!data) {
    throw new Error('data must be an Object.')
  } else if (!data.id) {
    throw new Error('`id` is unvalidated.')
  } else if (!data.type && !data.component) {
    throw new Error('`type` and `component` cannot both be null.')
  }
}

export default {
  components: {
    /**
     * 🐂🍺只需要有组件选项对象，就可以立刻包装成函数式组件在 template 中使用
     * FYI: https://cn.vuejs.org/v2/guide/render-function.html#%E5%87%BD%E6%95%B0%E5%BC%8F%E7%BB%84%E4%BB%B6
     */
    CustomComponent: {
      functional: true,
      render: (h, ctx) => h(ctx.props.component, ctx.data, ctx.children),
    },
    searchSelectInput,
  },
  props: {
    data: {
      type: Object,
      default() {
        return null
      },
    },
    prop: {
      type: String,
      default() {
        return this.data.id
      },
    },
    itemValue: {},
    value: Object,
    disabled: Boolean,
    readonly: Boolean,
    options: Array,
  },
  data() {
    return {
      propsInner: {},
      isBlurTrigger:
        this.data.rules &&
        this.data.rules.some(rule => {
          return rule.required && rule.trigger === 'blur'
        }),
    }
  },
  computed: {
    // 解构运算符会处理 undefined 的情况
    componentProps: ({data: {el}, propsInner}) => ({...el, ...propsInner}),
    hasReadonlyContent: ({data: {type}}) =>
      _includes(['input', 'select'], type),
    hiddenStatus: ({data: {hidden = () => false}, data, value}) =>
      hidden(value, data),
    enableWhenStatus: ({data: {enableWhen}, value}) =>
      getEnableWhenStatus(enableWhen, value),
    // 是否显示
    _show() {
      return !this.hiddenStatus && this.enableWhenStatus
    },
    listeners() {
      const {
        data: {
          id,
          type,
          atChange = noop,
          on = {},
          on: {input: originOnInput = noop, change: originOnChange = noop} = {},
          trim = true,
        },
        $parent: {
          $parent: {updateForm},
        },
      } = this
      return {
        ..._frompairs(
          _topairs(on).map(([eName, handler]) => [
            eName,
            (...args) => handler(args, updateForm),
          ]),
        ),
        // 手动更新表单数据
        input: (value, ...rest) => {
          this.$emit('updateValue', {id, value})
          // 更新表单时调用
          atChange(id, value)
          originOnInput([value, ...rest], updateForm)

          // FIXME: rules 的 trigger 只写了 blur，依然会在 input 的时候触发校验！
          this.triggerValidate(id)
        },
        change: (value, ...rest) => {
          if (typeof value === 'string' && trim) value = value.trim()
          if (type === 'select-input') {
            this.$emit('updateValue', {
              id,
              value: {
                id: value,
                value: rest[0],
              },
            })
            originOnChange([rest[0], []], updateForm)
          } else {
            this.$emit('updateValue', {id, value})
            originOnChange([value, ...rest], updateForm)

            // FIXME: rules 的 trigger 只写了 blur，依然会在 change 的时候触发校验！
          }
          this.triggerValidate(id)
        },
      }
    },

    multipleValue: ({data, itemValue, options = []}) => {
      const multipleSelectValue =
        _get(data, 'el.multiple') && Array.isArray(itemValue)
          ? itemValue
          : [itemValue]
      return multipleSelectValue
        .map(val => (options.find(op => op.value === val) || {}).label)
        .join()
    },
  },
  watch: {
    data: validator,
    /**
     * 这里其实用 remote 处理了两件事。有机会是可以拆分的
     * 1. 基本用法，配置 url 后即可从远程获取某个 prop 注入到组件
     * 2. 针对 select、checkbox-group & radio-group 组件，会直接将 resp 作为 options 处理；label & value 也是直接为这个场景而生的
     */
    'data.remote': {
      handler(v, oldV) {
        if (!v) return
        if (oldV) {
          if (v.url === oldV.url || v.request === oldV.request) return
        }
        const isOptionsCase =
          ['select', 'checkbox-group', 'radio-group'].indexOf(this.data.type) >
          -1
        const {
          url,
          request = () => this.$axios.get(url).then(resp => resp.data),
          prop = 'options', // 默认处理 el-cascader 的情况
          dataPath = '',
          onResponse = resp => {
            if (dataPath) resp = _get(resp, dataPath)
            if (isOptionsCase) {
              return resp.map(item => ({
                label: item[label],
                value: item[value],
              }))
            } else {
              return resp
            }
          },
          onError = error => console.error(error.message),
          label = 'label',
          value = 'value',
        } = v
        Promise.resolve(request())
          .then(onResponse, onError)
          .then(resp => {
            if (isOptionsCase) {
              let formRenderer = this.$parent
              while (formRenderer.$options._componentTag !== 'el-form-renderer')
                formRenderer = formRenderer.$parent
              formRenderer.setOptions(this.prop, resp)
            } else {
              this.propsInner = {[prop]: resp}
            }
          })
      },
      immediate: true,
    },
  },
  methods: {
    triggerValidate(id) {
      if (!this.data.rules || !this.data.rules.length) return

      /**
       * 下面代码可参考 `emitter`
       * 目的: 为了清除表单校验信息
       * 因为有部分表单项的值变更时没有清除校验信息, 因此需要触发一次校验用于清除
       * https://github.com/ElemeFE/element/blob/dev/src/mixins/emitter.js
       */
      let parent = this.$parent
      let name = parent.$options.componentName

      while (parent && name !== 'ElForm') {
        parent = parent.$parent

        if (parent) {
          name = parent.$options.componentName
        }
      }

      if (!parent || this.isBlurTrigger) return

      this.$nextTick(() => {
        parent.validateField(id)
      })
    },
  },
}
</script>
<style lang="less">
.render-form-item {
  textarea[readonly] {
    border: 1px solid #dcdee6; // 只读状态不要有 focus 状态
  }

  input[readonly] {
    border: 1px solid #dcdee6;
  }
}
</style>
