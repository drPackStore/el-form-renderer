<template>
  <!-- search下拉选择字段&输入组件 -->
  <div class="search-select-input">
    <el-select
      v-model="selectValue"
      class="search-select-input__select"
      placeholder="请选择"
      @change="selectChange"
    >
      <el-option
        v-for="item in options"
        :key="item.value"
        :label="item.label"
        :value="item.value"
      />
    </el-select>
    <el-input
      v-model="inputValue"
      class="search-select-input__input"
      placeholder="请输入"
      @change="inputChange"
    />
  </div>
</template>

<script>
export default {
  props: {
    options: {
      type: Array,
      default: () => [],
    },
    value: {
      type: Object,
      default: () => {},
    },
  },
  data() {
    return {
      selectValue: (this.value && this.value.id) || this.options[0].value,
      inputValue: (this.value && this.value.value) || '',
    }
  },
  watch: {
    value(val) {
      if (!val) {
        this.selectValue = this.options[0].value
        this.inputValue = ''
      } else {
        this.selectValue = val.id
        this.inputValue = val.value
      }
    },
  },
  methods: {
    inputChange(val) {
      this.$emit('change', this.selectValue, val)
    },
    selectChange(val) {
      this.$emit('change', val, this.inputValue)
    },
  },
}
</script>
<style lang="less" scoped>
.search-select-input {
  display: flex;

  &__select {
    width: 120px;

    /deep/ input {
      border-top-right-radius: 0;
      border-bottom-right-radius: 0;
    }
  }

  &__input {
    width: 200px;

    /deep/ input {
      border-top-left-radius: 0;
      border-bottom-left-radius: 0;
      border-left: none;
    }
  }
}
</style>
