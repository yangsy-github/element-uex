## 参数操作控件

参数操作控件。

### 基本用法


::: demo 参数操作控件
```html
<elx-operate-param v-model="testData"></elx-operate-param>
<script>
  export default {
    data: function() {
      return {
        testData: [{paramId: '', paramValue: ''}]
      }
    },
    methods: {
      changeData: function(val){
          //console.log(val)
      }
    },
    watch: {
      testData: {
        deep: true,
        handler: function(val, oldVal) {
          console.log(val);
        }
      }
    }
  }
</script>

```
:::

### Attributes
| 参数      | 说明          | 类型      | 可选值                           | 默认值  |
|---------- |-------------- |---------- |--------------------------------  |-------- |
| value | 数据绑定值 | Array | - | [] |
| is-add | 是否显示添加按钮 | Boolean | －| true |
| is-remove | 是否显示删除按钮 | Boolean | － | true |
| param-id-disable | 是否禁用参数名 | Boolean | － | false |

### Events
| 事件名称 | 说明 | 回调参数 |
|---------- |-------- |---------- |
| change | 绑定值变化时触发的事件 | - |
