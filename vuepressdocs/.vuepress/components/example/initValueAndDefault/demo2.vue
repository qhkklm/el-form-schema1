  <template>
  <div>
    <p>{{ model }}</p>
    <el-form-schema
      :schema="schema"
      v-model="model"
      :inline="false"
      ref="efs"
      label-width="150px"
    >
    <el-form-item>
        <el-button type="primary" @click="submit">提交</el-button>
        <el-button @click="reset">重置 (恢复到默认值)</el-button>
      </el-form-item>
    </el-form-schema>
  </div>
</template>
<script>
export default {
  data() {
    return {
      schema: {
        select: {
          tag: "el-select",
          label: "下拉分组🌟",
          group: { label: 'label', children: 'options' },
          slot: { after: "注意设置：group: { label: 'label', children: 'options' }" },
          default: "Shanghai",
          items:  [{
            label: '热门城市',
            options: [{
              value: 'Shanghai',
              label: '上海'
            }, {
              value: 'Beijing',
              label: '北京'
            }]
          }]
        },
        checkbox: {
          tag: "el-checkbox",
          label: "复选框",
          items: ["兴盛优选", "美团优选", "多多买菜"],
          default: ["兴盛优选", "美团优选"]
        },
        daterange: {
          tag: "el-date-picker",
          label: "日期范围",
          default: "2022-03-08"
        },
        obj1: {
          tag: "object",
          label: "对象",
          required: true,
          type: 'card',
          title: "测试",
          labelWidthComponents: '60px',
          components: {
            input: { 
              tag: "el-input", 
              label: "测试", 
              inline: true, 
              required: true, 
              default: "hello",
              slot: { after: " " }
            },
            radio: {
              tag: "el-radio", 
              inline: true, 
              items: this.arrayData(4),
              default: 2, 
            },
            checkbox: {
              tag: "el-checkbox",
              label: "复选框",
              items: ["兴盛优选", "美团优选", "多多买菜"],
              default: ["兴盛优选"]
            },
          }
        }
      },
      model: {}
    };
  },
  methods: {
    submit() {
      this.$refs.efs.validate(valid => {
        alert(valid);
      });
    },
    reset() {
      this.$refs.efs.resetFields();
    },
    arrayData(num) {
      return new Array(num)
        .fill({})
        .map((item, index) => ({ label: `测试-${index}`, value: index + 1 }));
    },
  }
};
</script>
