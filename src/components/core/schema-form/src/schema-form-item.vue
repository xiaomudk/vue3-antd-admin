<template>
  <Col v-if="getShow.isIfShow" v-show="getShow.isShow" v-bind="schema.colProps">
    <Divider v-if="schema.component === 'Divider'" v-bind="getComponentProps">
      <component :is="renderLabelHelpMessage"></component>
    </Divider>
    <Form.Item
      v-else
      v-bind="{ ...schema.formItemProps }"
      :label="renderLabelHelpMessage"
      :name="schema.field"
      :label-col="itemLabelWidthProp.labelCol"
      :wrapper-col="itemLabelWidthProp.wrapperCol"
      :rules="getRules"
    >
      <slot v-if="schema.slot" :name="schema.slot" v-bind="getValues"> </slot>
      <component
        :is="getComponent"
        v-else-if="getComponent"
        :ref="setItemRef"
        :key="schema.field"
        v-bind="getComponentProps"
        v-model:[modelValueType]="modelValue[schema.field]"
        :allow-clear="true"
        :disabled="getDisable"
        v-on="componentEvents"
      >
        <template v-if="Object.is(schema.loading, true)" #notFoundContent>
          <Spin size="small" />
        </template>
        <template
          v-for="(slotFn, slotName) in getComponentSlots"
          #[slotName]="slotData"
          :key="slotName"
        >
          <component
            :is="slotFn?.({ ...getValues, slotData }) ?? slotFn"
            :key="slotName"
          ></component>
        </template>
      </component>
    </Form.Item>
  </Col>
</template>

<script setup lang="tsx">
  import { computed, unref, toRefs, onMounted, isVNode } from 'vue';
  import { useVModel, isFunction } from '@vueuse/core';
  import { cloneDeep } from 'lodash-es';
  import { Form, Col, Spin, Divider } from 'ant-design-vue';
  import { useItemLabelWidth } from './hooks/useLabelWidth';
  import { componentMap } from './componentMap';
  import { createPlaceholderMessage } from './helper';
  import { useFormContext } from './hooks/useFormContext';
  import type { PropType } from 'vue';
  import type { ComponentMapType } from './componentMap';
  import type { CustomRenderFn, FormSchema, RenderCallbackParams } from './types/form';
  import type { RuleObject } from 'ant-design-vue/es/form/';
  import type { TableActionType } from '@/components/core/dynamic-table';
  import { isBoolean, isNull, isObject, isString } from '@/utils/is';
  import BasicHelp from '@/components/basic/basic-help/index.vue';
  import { useI18n } from '@/hooks/useI18n';

  defineOptions({
    name: 'SchemaFormItem',
  });

  const props = defineProps({
    formModel: {
      type: Object as PropType<Record<string, any>>,
      default: () => ({}),
    },
    schema: {
      type: Object as PropType<FormSchema>,
      default: () => ({}),
    },
    setFormModel: {
      type: Function as PropType<(key: string, value: any) => void>,
      default: null,
    },
    // 动态表格实例
    tableInstance: {
      type: Object as PropType<TableActionType>,
    },
    /** 将表单组件实例保存起来 */
    setItemRef: {
      type: Function,
      default: () => ({}),
    },
  });

  const emit = defineEmits(['update:formModel']);

  // schemaForm组件实例
  const formContext = useFormContext();
  const { formPropsRef } = formContext;

  const modelValue = useVModel(props, 'formModel', emit);
  const { t } = useI18n();

  const { schema } = toRefs(props);

  // @ts-ignore
  const itemLabelWidthProp = useItemLabelWidth(schema, formPropsRef);

  const modelValueType = computed(() => {
    const { component, componentProps } = schema.value;
    if (!isFunction(componentProps) && componentProps?.vModelKey) {
      return componentProps.vModelKey;
    }
    const isCheck = isString(component) && ['Switch', 'Checkbox'].includes(component);
    return isCheck ? 'checked' : 'value';
  });

  const getValues = computed<RenderCallbackParams>(() => {
    const { formModel, schema, tableInstance } = props;

    const { mergeDynamicData } = unref(formPropsRef);
    return {
      field: schema.field,
      formInstance: formContext,
      tableInstance,
      formModel,
      values: {
        ...mergeDynamicData,
        ...formModel,
      } as Recordable,
      schema,
    };
  });

  const getShow = computed<{ isShow: boolean; isIfShow: boolean }>(() => {
    const { vShow, vIf, isAdvanced = false } = unref(schema);
    const { showAdvancedButton } = unref(formPropsRef);
    const itemIsAdvanced = showAdvancedButton ? (isBoolean(isAdvanced) ? isAdvanced : true) : true;

    let isShow = true;
    let isIfShow = true;

    if (isBoolean(vShow)) {
      isShow = vShow;
    }
    if (isBoolean(vIf)) {
      isIfShow = vIf;
    }
    if (isFunction(vShow)) {
      isShow = vShow(unref(getValues));
    }
    if (isFunction(vIf)) {
      isIfShow = vIf(unref(getValues));
    }
    isShow = isShow && itemIsAdvanced;

    return { isShow, isIfShow };
  });

  const getDisable = computed(() => {
    const { disabled: globDisabled } = unref(formPropsRef);
    const { dynamicDisabled } = props.schema;
    const { disabled: itemDisabled = false } = unref(getComponentProps);
    let disabled = !!globDisabled || itemDisabled;
    if (isBoolean(dynamicDisabled)) {
      disabled = dynamicDisabled;
    }
    if (isFunction(dynamicDisabled)) {
      disabled = dynamicDisabled(unref(getValues));
    }
    return disabled;
  });

  const vnodeFactory = (
    component: FormSchema['componentSlots'] | FormSchema['component'],
    values = unref(getValues),
  ) => {
    if (isString(component)) {
      return <>{component}</>;
    } else if (isVNode(component)) {
      return component;
    } else if (isFunction(component)) {
      return vnodeFactory((component as CustomRenderFn)(values));
    } else if (isObject(component)) {
      const compKeys = Object.keys(component);
      // 如果是组件对象直接return
      if (compKeys.some((n) => n.startsWith('_') || ['setup', 'render'].includes(n))) {
        return component;
      }
      return compKeys.reduce<Recordable<CustomRenderFn>>((slots, slotName) => {
        slots[slotName] = (...rest: any) => vnodeFactory(component[slotName], rest);
        return slots;
      }, {});
    }
    return component;
  };

  /**
   * @description 当前表单项组件
   */
  const getComponent = computed(() => {
    const component = props.schema.component;
    return isString(component)
      ? componentMap[component] ?? vnodeFactory(component)
      : vnodeFactory(component);
  });

  /**
   * @description 当前表单项组件的插槽
   */
  const getComponentSlots = computed<Recordable<CustomRenderFn>>(() => {
    const componentSlots = props.schema.componentSlots ?? {};
    return isString(componentSlots) || isVNode(componentSlots)
      ? {
          default: (...rest: any) => vnodeFactory(componentSlots, rest),
        }
      : vnodeFactory(componentSlots);
  });

  /**
   * @description 表单组件props
   */
  const getComponentProps = computed(() => {
    const { schema } = props;
    let { componentProps = {}, component, label = '' } = schema;

    if (isFunction(componentProps)) {
      componentProps = componentProps(unref(getValues)) ?? {};
    }

    if (component !== 'RangePicker' && isString(component)) {
      componentProps.placeholder ??= createPlaceholderMessage(component, label);
    }
    if (schema.component === 'Divider') {
      componentProps = Object.assign({ type: 'horizontal' }, componentProps, {
        orientation: 'left',
        plain: true,
      });
    }
    schema.field === 'field35' && console.log('componentProps', componentProps);

    return componentProps as Recordable;
  });

  /**
   * @description 表单组件事件
   */
  const componentEvents = computed(() => {
    const componentProps = props.schema?.componentProps || {};
    return Object.keys(componentProps).reduce((prev, key) => {
      if (/on([A-Z])/.test(key)) {
        // eg: onChange => change
        const eventKey = key.replace(/on([A-Z])/, '$1').toLocaleLowerCase();
        prev[eventKey] = componentProps[key];
      }
      return prev;
    }, {});
  });

  const renderLabelHelpMessage = computed(() => {
    const { label, helpMessage, helpComponentProps, subLabel } = props.schema;
    const renderLabel = subLabel ? (
      <span>
        {label} <span class="text-secondary">{subLabel}</span>
      </span>
    ) : (
      vnodeFactory(label)
    );
    const getHelpMessage = isFunction(helpMessage) ? helpMessage(unref(getValues)) : helpMessage;
    if (!getHelpMessage || (Array.isArray(getHelpMessage) && getHelpMessage.length === 0)) {
      return renderLabel;
    }
    return (
      <span>
        {renderLabel}
        <BasicHelp placement="top" class="mx-1" text={getHelpMessage} {...helpComponentProps} />
      </span>
    );
  });

  function setComponentRuleType(
    rule: RuleObject,
    component: ComponentMapType,
    valueFormat: string,
  ) {
    if (['DatePicker', 'MonthPicker', 'WeekPicker', 'TimePicker'].includes(component)) {
      rule.type = valueFormat ? 'string' : 'object';
    } else if (['RangePicker', 'Upload', 'CheckboxGroup', 'TimePicker'].includes(component)) {
      rule.type = 'array';
    } else if (['InputNumber'].includes(component)) {
      rule.type = 'number';
    }
  }

  const getRules = computed(() => {
    const {
      rules: defRules = [],
      component,
      rulesMessageJoinLabel,
      label,
      dynamicRules,
      required,
    } = props.schema;

    if (isFunction(dynamicRules)) {
      return dynamicRules(unref(getValues)) as RuleObject[];
    }

    let rules = cloneDeep<RuleObject[]>(defRules);
    const { rulesMessageJoinLabel: globalRulesMessageJoinLabel } = unref(formPropsRef);

    const joinLabel = Reflect.has(unref(formPropsRef), 'rulesMessageJoinLabel')
      ? rulesMessageJoinLabel
      : globalRulesMessageJoinLabel;
    const defaultMsg = isString(component)
      ? `${createPlaceholderMessage(component, label)}${joinLabel ? label : ''}`
      : undefined;

    function validator(rule: any, value: any) {
      const msg = rule.message || defaultMsg;

      if (value === undefined || isNull(value)) {
        // 空值
        return Promise.reject(msg);
      } else if (Array.isArray(value) && value.length === 0) {
        // 数组类型
        return Promise.reject(msg);
      } else if (typeof value === 'string' && value.trim() === '') {
        // 空字符串
        return Promise.reject(msg);
      } else if (
        typeof value === 'object' &&
        Reflect.has(value, 'checked') &&
        Reflect.has(value, 'halfChecked') &&
        Array.isArray(value.checked) &&
        Array.isArray(value.halfChecked) &&
        value.checked.length === 0 &&
        value.halfChecked.length === 0
      ) {
        // 非关联选择的tree组件
        return Promise.reject(msg);
      }
      return Promise.resolve();
    }

    const getRequired = isFunction(required) ? required(unref(getValues)) : required;

    if ((!rules || rules.length === 0) && getRequired) {
      rules = [{ required: getRequired, validator }];
    }

    const requiredRuleIndex: number = rules.findIndex(
      (rule) => Reflect.has(rule, 'required') && !Reflect.has(rule, 'validator'),
    );

    if (requiredRuleIndex !== -1) {
      const rule = rules[requiredRuleIndex];

      if (component && isString(component)) {
        if (!Reflect.has(rule, 'type')) {
          rule.type = component === 'InputNumber' ? 'number' : 'string';
        }

        rule.message = rule.message || defaultMsg;

        if (component.includes('Input') || component.includes('Textarea')) {
          rule.whitespace = true;
        }
        const valueFormat = unref(getComponentProps)?.valueFormat;
        setComponentRuleType(rule, component, valueFormat);
      }
    }

    // Maximum input length rule check
    const characterInx = rules.findIndex((val) => val.max);
    if (characterInx !== -1 && !rules[characterInx].validator) {
      rules[characterInx].message =
        rules[characterInx].message ||
        t('component.form.maxTip', [rules[characterInx].max] as Recordable);
    }

    return rules;
  });

  onMounted(async () => {
    if (getComponentProps.value?.request) {
      const compProps = getComponentProps.value;
      const { componentProps, component } = schema.value;

      schema.value.loading = true;
      schema.value.field === 'field35' && console.log('compProps', compProps, formPropsRef.value);

      try {
        const result = await getComponentProps.value?.request(unref(getValues));
        if (['Select', 'RadioGroup', 'CheckBoxGroup'].some((n) => n === component)) {
          compProps.options = result;
        } else if (['TreeSelect', 'Tree'].some((n) => n === component)) {
          compProps.treeData = result;
        }
        if (componentProps) {
          componentProps.requestResult = result;
        }
      } finally {
        schema.value.loading = false;
      }
    }
  });
</script>
