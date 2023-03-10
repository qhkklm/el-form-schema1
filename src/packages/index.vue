<script>
import { Component } from './components/index'
import diffData from './components/diff'
import { getObjectByPath, isEqual, isEmpty, genUnique } from './components/utils'
export default {
  model: {
    prop: 'model',
    event: 'input'
  },
  props: {
    model: {
      type: Object,
      required: true,
      default: () => ({})
    },
    disabled: {
      type: Boolean,
      default: false
    },
    schema: {
      type: Object,
      required: true,
      default: () => ({})
    },
    isSearchForm: {
      type: Boolean,
      default: false
    },
    expandNumber: {
      type: Number,
      default: 0
    },
    labelPosition: {
      type: String,
      default() {
        return this.$EFS.labelPosition || 'right'
      }
    },
    size: {
      type: String,
      default() {
        return this.$EFS.size || 'small'
      }
    },
    inline: {
      type: Boolean,
      default: false
    },
    labelSuffix: {
      type: String,
      default() {
        return this.$EFS.labelSuffix || ''
      }
    },
    labelWidth: {
      type: String,
      default: '0px'
    },
    componentWidth: {
      type: String,
      default() {
        return this.$EFS.componentWidth || '220px'
      }
    },
    type: {
      type: String,
      default: ''
    },
    refName: {
      type: String,
      default: () => {
        return `el-form-schema-${genUnique()}`
      }
    },
    submitProps: {
      type: Object,
      default() {
        return { type: 'primary', ...this.$EFS.submitProps }
      }
    },
    resetProps: {
      type: Object,
      default() {
        return { type: 'default', ...this.$EFS.resetProps }
      }
    },
    expandProps: {
      type: Object,
      default() {
        return { type: 'default', ...this.$EFS.expandProps }
      }
    },
    isExpand: {
      type: Boolean,
      default() {
        return this.$EFS.isExpand
      }
    },
    debug: {
      type: Boolean,
      default: false
    },
    apiConfig: {
      type: Function
    },
    useEnterSearch: {
      type: Boolean,
      default: true
    }
  },
  data() {
    return {
      schemaValues: {},
      configData: {},
      expandAll: this.isExpand,
      isWatching: false,
      validateFieldSet: new Set(),
      enterFormValues: {},
      formValues: {},
      initValues: {}
    }
  },
  computed: {
    isCanExpand() {
      return (this.expandNumber > 0 && Object.keys(this.schema).length > this.expandNumber)
    }
  },
  watch: {
    formValues: {
      handler(val) {
        this.isWatching = true
        this.$emit('input', val)
      },
      deep: true
    },
    model: {
      handler(val) {
        this.$nextTick(() => {
          Object.assign(this.formValues, val)
        })
      },
      deep: true,
      immediate: true
    },
    schema: {
      handler(val) {
        if (!this.isWatching && Object.keys(val).length) {
          this.initSchemas()
        } else {
          this.isWatching = false
        }
      },
      // ????????????????????????????????????render???????????????bug
      // deep: true,
      immediate: true
    }
  },
  created() {
    if (typeof this.apiConfig === 'function') {
      this.apiConfig().then(data => {
        Object.keys(data).map(key => {
          this.$set(this.configData, key, data[key])
        })
      })
    }
  },
  updated() {
    // ??????vif??????false???????????????????????????
    Array.from(this.validateFieldSet).forEach(field => {
      this.clearValidate(field)
    })
    // ???????????????????????? class = el-form-item-inline????????????????????? el-form ??? inline ??????
    Array.from(document.querySelectorAll('.el-form-item-inline')).forEach(el => {
      el.querySelector('.el-form-item__content').style.marginLeft = '0px'
      // ?????? slot ??? label-width
      const slotLabelWidth = el.getAttribute('slot-label-width')
      if (el.querySelector('.is-required') && slotLabelWidth) {
        el.querySelector('.el-form-item__content').style.marginLeft = slotLabelWidth
        el.querySelector('.el-form-item').style.marginBottom = '22px'
      }
      if (slotLabelWidth && el.querySelector('.el-form-item__label')) {
        el.querySelector('.el-form-item__label').style.width = slotLabelWidth
      }
    })
    // ???????????????????????? class = is-set-inline??????????????? inline ??????
    Array.from(document.querySelectorAll('.is-set-inline')).forEach(el => {
      if (el.querySelector('.el-form-item__label')) {
        el.querySelector('.el-form-item__content').style.marginLeft = '0px'
      }
      if (el.querySelector('.el-form-item') && !el.querySelector('.el-form-item').style.display) {
        el.querySelector('.el-form-item').style.display = 'inline-flex'
      }
    })
  },
  activated() {
    // ?????? keep-alive ?????????bug
    this.onEnterSearch()
  },
  mounted() {
    this.onEnterSearch()
  },
  methods: {
    /**
		 * @description: ???????????????schema
		*/
    async initSchemas() {
      const values = {}
      const lastKey = Object.keys(this.schema).pop()
      for (const key in this.schema) {
        this.schema[key].$isLast = lastKey === key
        const schemaComponent = this.schema[key]
        this.initComponentList(schemaComponent)
        this.setValueKey(values, key, schemaComponent)
      }
      const customData = Object.keys(this.model).reduce((prev, key) => {
        if (!(key in values)) prev[key] = this.model[key]
        return prev
      }, {})
      this.schemaValues = JSON.parse(JSON.stringify(values))
      this.formValues = { ...values, ...customData }
      this.$emit('input', { ...this.formValues, ...this.model })
      await this.$nextTick()
      this.diffDataByInitValues(this.initValues, this.formValues)
    },
    /**
		 *  ?????? diff ???????????????????????????????????????
		*/
    diffDataByInitValues(current, previous) {
      const diffResult = diffData(current, previous)
      const getModelByPath = (arrPath) => arrPath.reduce((cur, prev) => cur[prev], previous)
      Object.keys(diffResult).map(path => {
        const arrPath = path.substr(0, path.lastIndexOf('.')).split('.').filter(Boolean)
        const targetKey = path.substr(path.lastIndexOf('.') + 1)
        const prevModel = arrPath.length > 0 ? getModelByPath(arrPath) : previous
        if (diffResult[path] && !prevModel[targetKey]) {
          if (targetKey.includes('[')) {
            const [key, index] = targetKey.replace(']', '').split('[')
            this.$set(prevModel[key], index, diffResult[path])
          } else {
            this.$set(prevModel, targetKey, diffResult[path])
          }
        }
      })
    },
    /**
		 *  label/title/slot ???????????????
		*/
    setExpTpl(component) {
      // slot: { after: "???????????????", prepend: "???????????????", append: "???????????????" } ??? slot: "???????????????"
      if (typeof component.slot === 'string' && /\$\{.+?\}/g.test(component.slot)) {
        component['slot_template_string'] = component.slot
      }
      if (typeof component.slot === 'object' && /\$\{.+?\}/g.test(component.slot.after)) {
        component['slot_after_template_string'] = component.slot.after
      }
      if (typeof component.slot === 'object' && /\$\{.+?\}/g.test(component.slot.prepend)) {
        component['slot_prepend_template_string'] = component.slot.prepend
      }
      if (typeof component.slot === 'object' && /\$\{.+?\}/g.test(component.slot.append)) {
        component['slot_append_template_string'] = component.slot.append
      }
      // title/label ???????????????
      ['title', 'label'].forEach(key => {
        const expTagProp = `${key}_template_string`
        const val = component[key]
        if (/\$\{.+?\}/g.test(val)) {
          component[expTagProp] = val
        }
      })
    },
    /**
		 *  ???????????????:  props ??? attrs
		*/
    setExp(component) {
      // ?????? required: '$model.a'
      if (component.required && typeof component.required === 'string') {
        component.requiredExpression = component.required
      }
      // ?????? inline: '$model.a'
      if (component.inline && typeof component.inline === 'string') {
        component.inlineExpression = component.inline
      }
      // ?????? rules: { required: '$model.a', message: '??????' }
      if (!Array.isArray(component.rules) && component.rules && (typeof component.rules.required === 'string')) {
        component.requiredExpression = component.rules.required
      }
      // ?????? props ?????????
      Object.keys(component.props || {}).map(key => {
        const expProp = `${key}_exp_prop`
        const val = component.props[key]
        if (/\$index|\$item|\$model/g.test(val)) {
          component.props[expProp] = 	val
        }
      })
      // ?????? attrs ?????????
      Object.keys(component.attrs || {}).map(key => {
        const expAttr = `${key}_exp_attr`
        const val = component.attrs[key]
        if (/\$index|\$item|\$model/g.test(val)) {
          component.attrs[expAttr] = 	val
        }
      })
    },
    /**
		 * ?????????????????????: $watchConfig
		*/
    watchConfigExp(component, wrapComponent) {
      // ?????????????????????$watchConfig ??? $watchModel
      const watchTarget = (target, key) => {
        const targetExp = target[key]
        if (/\$config\./g.test(targetExp)) {
          const watchModel = targetExp.replace(/\$config/g, 'configData')
          target[key] = []
          this.$watch(watchModel, (val) => {
            target[key] = val
            // ????????????????????????????????? table/array ???items?????????????????????????????????bug
            this.$nextTick(() => {
              if (wrapComponent) {
                wrapComponent.refreshKey = `rk.${genUnique()}`
                this.$forceUpdate()
              }
            })
          })
        }
      }
      // (el-checkbox???el-select???el-radio) ?????? items: '$configData.test' ???????????????watch??????
      watchTarget(component, 'items')
    },
    /**
		 * @description: ?????????????????????
		*/
    initComponentDefaultProps(component, watchConfigExp) {
      component.isMarginBottom = true
      component.refreshKey = ''
      component.isInput = component.hasOwnProperty('isInput') ? component.isInput : true
      component.vif = component.hasOwnProperty('vif') ? component.vif : true
      component.$item = null
      component.$index = -1
      component.$isArrayLast = false
      component.label = component.label || ''
      component.style = component.style || {}
      component.slot = component.slot || {}
      component.items = component.items || []
      component.props = component.props || {}
      component.attrs = component.attrs || {}
      component.rules = component.rules || (component.required ? { required: true, message: '??????' } : null)
      // ??????config??????????????????
      this.watchConfigExp(component, watchConfigExp)
      switch (component.tag) {
        case 'p':
        case 'a':
        case 'i':
        case 'h1':
        case 'h2':
        case 'h3':
        case 'h4':
        case 'h5':
        case 'div':
        case 'span':
          component.isInput = false
          break
        case 'card':
          component.isInput = false
          break
        case 'el-divider':
          component.isInput = false
          Object(component.style, { margin: '16px 0' })
          break
        case 'el-input':
          component.attrs.placeholder = component.props.placeholder || `?????????${/\$\{.+?\}/g.test(component.label) ? '' : component.label}`
          break
        case 'el-date-picker':
          component.props = Object.assign({}, { placeholder: '???????????????', startPlaceholder: '????????????', endPlaceholder: '????????????', unlinkPanels: true }, component.props)
          if (component.props.type === 'daterange') {
            component.default = component.default || []
            component.props.valueFormat = component.props.valueFormat || 'yyyy-MM-dd'
          } else if (component.props.type === 'datetimerange') {
            component.default = component.default || []
            component.props.valueFormat = component.props.valueFormat || 'yyyy-MM-dd HH:mm:ss'
            component.props.defaultTime = component.props.defaultTime || ['00:00:00', '23:59:59']
          } else if (component.props.type === 'datetime') {
            component.default = component.default || ''
            component.props.valueFormat = component.props.valueFormat || 'yyyy-MM-dd HH:mm:ss'
          } else {
            component.default = component.default || ''
            component.props.valueFormat = component.props.valueFormat || 'yyyy-MM-dd'
          }
          break
        case 'el-switch':
          component.default = component.default || false
          break
        case 'el-cascader':
          component.default = component.default || []
          break
        case 'el-slider':
          component.default = component.default || 0
          break
        case 'el-time-picker':
        case 'el-time-select':
          component.props = Object.assign({ placeholder: '???????????????', startPlaceholder: '????????????', endPlaceholder: '????????????', valueFormat: 'HH:mm:ss' }, component.props)
          break
        case 'el-select':
        case 'el-radio':
        case 'el-checkbox':
          component.props.placeholder = component.props.placeholder || `?????????${/\$\{.+?\}/g.test(component.label) ? '' : component.label}`
          component.keys = Object.assign({ label: 'label', value: 'value' }, component.keys || {})
          if (component.tag === 'el-checkbox') {
            component.default = component.default || (component.type === 'bool' ? false : [])
            component.checkAll = component.checkAll || []
            component.checkAllDisabled = component.checkAllDisabled || false
          }
          if (component.props.multiple) {
            component.default = component.default || []
          }
          break
        case 'object':
          component.inline = component.inline || false
          break
        case 'table':
        case 'array':
          component.minLimit = component.minLimit || 0
          component.maxLimit = component.maxLimit || 0
          component.list = component.list || []
          component.showValidate = false
          break
        default:
          // ????????????
          break
      }
      if (!component.style.width && component.isInput) {
        component.style.width = this.componentWidth
      }
    },
    /**
		 * @description: ????????????schema???????????????
		*/
    initComponentList(schema) {
      if (schema.components) {
        for (const _key in schema.components) {
          if (!('inline' in schema.components[_key])) {
            this.$set(schema.components[_key], 'inline', schema.inline)
          }
          if (schema.components[_key].components) {
            this.initComponentList(schema.components[_key], _key)
          } else {
            this.initComponentDefaultProps(schema.components[_key], schema)
          }
        }
      } else {
        this.initComponentDefaultProps(schema, null)
      }
    },
    /**
		 * @description: ????????????schema??????????????????v-model???key
		*/
    setValueKey(values, key, schema) {
      switch (schema.tag) {
        case 'object':
          if (schema.components) {
            schema.skip = !schema.hasOwnProperty('skip') ? false : schema.skip
            if (!schema.skip) {
              values[key] = {}
              this.initValues[key] = {}
            }
            schema.isMarginBottom = false
            schema.vifBool = true
            schema.componentWidth = schema.componentWidth || this.componentWidth
            this.setExpTpl(schema)
            if (schema.type === 'card' && !schema.hasOwnProperty('border')) schema.border = true
            for (const _key in schema.components) {
              schema.components[_key].isMarginBottom = true
              //
              this.setExpTpl(schema.components[_key])
              //
              if (!schema.skip) {
                // ????????????????????????
                if (schema.components[_key].initValue) {
                  this.initValues[key][_key] = schema.components[_key].initValue
                  delete schema.components[_key].initValue
                }
                this.setValueKey(values[key], _key, schema.components[_key])
              } else {
                this.setValueKey(values, _key, schema.components[_key])
              }
            }
          }
          break
        case 'table':
        case 'array':
          // eslint-disable-next-line no-case-declarations
          const keys = {}
          this.setExpTpl(schema)
          schema.vifBool = true
          schema.operator = typeof schema.operator === 'boolean' ? schema.operator : {}
          Object.keys(schema.components).forEach((_key) => {
            this.setExpTpl(schema.components[_key])
            if (schema.components[_key].tag !== 'action') {
              this.setValueKey(keys, _key, schema.components[_key])
            }
          })
          Object.assign(keys, schema.addRowExt || {})
          // eslint-disable-next-line no-prototype-builtins
          if (!schema.hasOwnProperty('keys')) {
            schema.keys = keys
          }
          values[key] = schema.default || []
          break
        default:
          this.setExpTpl(schema)
          if (schema.isInput) {
            // ?????????????????????
            if (schema.destruct) {
              schema.destruct.forEach(field => {
                values[field] = ''
              })
            }
            // ??????????????????
            if (schema.initValue) {
              this.initValues[key] = schema.initValue
            }
            values[key] = this.setDefaultValue(schema)
            this.setExp(schema)
            // ??????slot?????????
            if (typeof schema.slot === 'object') {
              Object.keys(schema.slot).forEach(key => {
                const obj = schema.slot[key]
                if (obj instanceof Object && obj['vmodel']) {
                  schema.slot[key].isInput = false
                  if (key === 'after') {
                    schema.slot[key].style = schema.slot[key].style || {}
                    schema.slot[key].inline = true
                    schema.slot[key].style.marginBottom = obj.marginBottom || '0px'
                  }
                  this.initComponentDefaultProps(obj)
                  if (obj.items) {
                    const list = JSON.parse(JSON.stringify(obj.items))
                    schema.slot[key].items = list.map(item => {
                      return (typeof item === 'string') ? { label: item, value: item } : item
                    })
                  }
                  values[obj.vmodel] = this.setDefaultValue(obj)
                  this.setExp(obj)
                }
              })
            }
          }
      }
    },
    /**
		 * @description: ???????????????
		*/
    setDefaultValue(item) {
      if (!isEmpty(item.default)) {
        return item.default
      } else {
        return ''
      }
    },
    /**
		 * @description: ?????? <el-form-item> <?????? v-model> </el-form-item>
		*/
    renderFormItem(h, item, key, isInput = true) {
      return Component(h, this, key, item, isInput)
    },
    /**
		 * @description: ?????? schema ???????????? el-form-item
		*/
    renderFormItems(h) {
      return Object.keys(this.schema).map((key, index) => {
        // key??????name?????????
        this.schema[key].name = key
        // ????????????/??????
        this.schema[key].expand = this.expandNumber > 0 ? index < this.expandNumber : true

        if (this.schema[key].tag === 'slot') {
          //  ????????????slot
          return this.$slots[this.schema[key].slot]
        } else {
          // ??????formItem
          return this.renderFormItem(h, this.schema[key], key)
        }
      })
    },
    /**
     *  ?????????????????????????????????????????????????????????
     */
    scrollToError(el, scrollOption = {
      behavior: 'smooth',
      block: 'center'
    }) {
      const validaErrDoms = el.getElementsByClassName('is-error') || []
      validaErrDoms.length && validaErrDoms[0].scrollIntoView(scrollOption)
    },
    /**
		 *  @description ??????????????????
		 */
    validate(cb) {
      const vm = this
      return this.$refs[this.refName].validate(async function() {
        cb.apply(this, arguments)
        await vm.$nextTick()
        try {
          vm.scrollToError(vm.$refs[vm.refName].$el)
        } catch (ex) {
          console.error('catch scrollToError error???', ex)
        }
      })
    },
    /**
		 * @description: ??????????????????????????????
		*/
    getValidateProps(field) {
      const val = getObjectByPath(this.formValues, field)
      if ((typeof val !== 'object' && !Array.isArray(val)) || val === null) {
        return [field]
      } else if (Array.isArray(val)) {
        return val.length === 0 ? [field] : val.map((item, index) => this.getValidateProps(`${field}[${index}]`)).flat(Infinity)
      } else {
        return Object.keys(val).reduce((prev, next) => {
          if (!next.includes('$key')) {
            if (typeof val[next] !== 'object' && !Array.isArray(val[next])) { // ????????????
              prev.push(`${field}.${next}`)
            } else if (Array.isArray(val[next])) { // ????????????
              val[next].length === 0 ? prev.push(`${field}.${next}`) : val[next].map((item, index) => prev.push(...this.getValidateProps(`${field}.${next}[${index}]`))).flat(Infinity)
            } else { // ????????????
              prev.push(...this.getValidateProps(`${field}.${next}`))
            }
          }
          return prev
        }, [])
      }
    },
    /**
		 * @description: ??????????????????
		*/
    validateField(props, cb) {
      props = this.getValidateProps(props)
      return this.$refs[this.refName] && this.$refs[this.refName].validateField(props, cb)
    },
    /**
		 * @description: ??????????????????
		*/
    validateFieldPromise(props) {
      props = this.getValidateProps(props)
      return Promise.all(props.map((item) =>
        new Promise((resolve) => {
          this.$refs[this.refName].validateField(item, valid => resolve(valid))
        })
      )).then(res => {
        return res.filter(item => item).length === 0
      })
    },
    /**
		 * @description: ??????????????????
		 */
    resetFields() {
      try {
        // ???????????????????????????
        this.validateFieldSet.clear()
        // ????????????
        this.$refs[this.refName].resetFields()
      } catch (ex) {
        // ?????????????????????????????????????????????????????????????????????????????????????????????????????????
        // Error: please transfer a valid prop path to form item!
      }
    },
    /**
		 * @description: ???????????? (????????????????????????????????????????????????)
		 */
    forceReset() {
      // ??????????????????array/table?????????minLimit ?????????????????????????????????Bug
      this.$nextTick(() => {
        this.$set(this, 'formValues', { ...this.model, ...JSON.parse(JSON.stringify(this.schemaValues)) })
        this.$nextTick(() => {
          this.$forceUpdate()
          this.$refs[this.refName].clearValidate()
        })
      })
      this.resetFields()
    },
    /**
		 * @description: ??????????????????
		 */
    clearValidate(props = []) {
      props = this.getValidateProps(props)
      return this.$refs[this.refName] && this.$refs[this.refName].clearValidate(props)
    },
    /**
    * @description: ??????enter????????????????????????
    */
    onEnterSearch() {
      if (this.isSearchForm && this.useEnterSearch) {
        document.onkeyup = (event) => {
          // eslint-disable-next-line no-caller
          const e = event || window.event || arguments.callee.caller.arguments[0]
          if (e && e.keyCode === 13) {
            const nodePaths = e.path.map(node => node.className)
            // ???????????????/????????????????????????enter????????????????????????
            if (!nodePaths.includes('el-dialog') && !nodePaths.includes('el-pagination__jump')) {
              if (!isEqual(this.formValues, this.enterFormValues) && this.$refs[this.refName]) {
                this.enterFormValues = JSON.parse(JSON.stringify(this.formValues))
                this.$refs[this.refName].validate((valid) => {
                  valid && this.$emit('submit', valid)
                })
              }
            }
          }
        }
      }
    }
  },
  render(h) {
    const vm = this

    if (Object.keys(vm.formValues).length === 0) return [h('template', null)]

    // ????????????
    const operatorButton = h('el-form-item', {
      props: {
        labelWidth: vm.labelWidth,
        size: vm.size,
        label: ' '
      }
    }, [
      h('el-button', {
        props: { ...vm.submitProps },
        on: {
          click() {
            vm.$refs[vm.refName].validate((valid) => {
              if (valid) {
                vm.$emit('submit', valid)
              }
            })
          }
        }
      }, '??????'),
      h('el-button', {
        props: { ...vm.resetProps },
        on: {
          click() {
            vm.resetFields()
            vm.$emit('reset')
          }
        }
      }, '??????'),
      vm.isCanExpand ? h(
        'el-button',
        {
          style: { display: !vm.expandAll ? '' : 'none' },
          props: { size: vm.size, ...vm.expandProps },
          on: {
            click() {
              vm.expandAll = !vm.expandAll
              vm.$emit('expand', vm.expandAll)
            }
          }
        },
        [
          '??????',
          h('i', {
            class: {
              'el-icon-caret-bottom': true
            }
          })
        ]
      ) : null,
      vm.isCanExpand ? h(
        'el-button',
        {
          style: { display: vm.expandAll ? '' : 'none' },
          props: { size: vm.size, ...vm.expandProps },
          on: {
            click() {
              vm.expandAll = !vm.expandAll
              vm.$emit('expand', vm.expandAll)
            }
          }
        },
        [
          '??????',
          h('i', {
            class: {
              'el-icon-caret-top': true
            }
          })
        ]
      ) : null
    ])

    const FormRender = h('el-form', {
      props: {
        model: vm.formValues,
        inline: vm.inline,
        labelWidth: vm.labelWidth,
        labelSuffix: vm.labelSuffix,
        size: vm.size,
        disabled: vm.disabled,
        labelPosition: vm.labelPosition
      },
      ref: vm.refName
    }, [
      ...(vm.renderFormItems(h) || []),
      ...(vm.$slots.default || []),
      vm.isSearchForm ? operatorButton : []
    ])
    // ??????
    return (vm.type === 'card') ? h('div', {
      style: { margin: '10px', border: '1px solid #dddddd', padding: '20px 20px 0 20px', borderShadow: '0 2px 12px 0 rgba(0,0,0,.1)' }
    }, [FormRender]) : FormRender
  }
}
</script>
