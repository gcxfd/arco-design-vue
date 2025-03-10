<template>
  <Trigger
    v-if="!hideTrigger"
    trigger="click"
    :click-to-close="false"
    :popup-translate="[0, 4]"
    v-bind="triggerProps"
    :unmount-on-close="unmountOnClose"
    :position="position"
    :disabled="triggerDisabled"
    :popup-visible="panelVisible"
    :popup-container="popupContainer"
    @popupVisibleChange="onPanelVisibleChange"
  >
    <slot>
      <DateRangeInput
        ref="refInput"
        v-bind="$attrs"
        v-model:focusedIndex="focusedIndex"
        :size="size"
        :focused="panelVisible"
        :visible="panelVisible"
        :error="error"
        :disabled="disabled"
        :editable="!readonly"
        :allow-clear="allowClear"
        :placeholder="computedPlaceholder"
        :input-value="inputValue"
        :value="panelValue"
        :format="computedFormat"
        @clear="onInputClear"
        @change="onInputChange"
        @pressEnter="onInputPressEnter"
      >
        <template #suffix-icon>
          <slot name="suffix-icon">
            <IconCalendar />
          </slot>
        </template>
        <template #separator>
          <slot name="separator">
            {{ separator || '-' }}
          </slot>
        </template>
      </DateRangeInput>
    </slot>
    <template #content>
      <RangePickerPanel v-bind="rangePanelProps" />
    </template>
  </Trigger>
  <RangePickerPanel v-else v-bind="{ ...$attrs, ...rangePanelProps }" />
</template>
<script lang="ts">
import { Dayjs } from 'dayjs';
import {
  computed,
  defineComponent,
  nextTick,
  PropType,
  reactive,
  ref,
  toRefs,
  watch,
} from 'vue';
import { TimePickerProps } from '../time-picker/interface';
import { DisabledTimeProps, RangePickerProps, ShortcutType } from './interface';
import { getPrefixCls } from '../_utils/global-config';
import { isArray } from '../_utils/is';
import pick from '../_utils/pick';
import { getFormattedValue, isValidInputValue } from '../time-picker/utils';
import {
  getSortedDayjsArray,
  isValueChange,
  dayjs,
  getNow,
  getDateValue,
} from '../_utils/date';
import useState from '../_hooks/use-state';
import {
  isCompleteRangeValue,
  isValidRangeValue,
  mergeValueWithTime,
} from './utils';
import useFormat from './hooks/use-format';
import useRangePickerState from './hooks/use-range-picker-state';
import useRangeHeaderValue from './hooks/use-range-header-value';
import Trigger from '../trigger';
import DateRangeInput from '../_components/picker/input-range.vue';
import RangePickerPanel from './range-picker-panel.vue';
import useRangeTimePickerValue from './hooks/use-range-time-picker-value';
import useIsDisabledDate from './hooks/use-is-disabled-date';
import { omit } from '../_utils/omit';
import useMergeState from '../_hooks/use-merge-state';
import IconCalendar from '../icon/icon-calendar';
import useProvideDatePickerTransform from './hooks/use-provide-datepicker-transform';

export default defineComponent({
  name: 'RangePicker',
  components: {
    RangePickerPanel,
    DateRangeInput,
    Trigger,
    IconCalendar,
  },
  inheritAttrs: false,
  props: {
    /**
     * @zh 范围选择器的类型
     * @en Type of range selector
     * */
    mode: {
      type: String as PropType<'date' | 'year' | 'quarter' | 'month' | 'week'>,
      default: 'date',
    },
    /**
     * @zh 绑定值
     * @en Value
     */
    modelValue: {
      type: Array as PropType<(Date | string | number)[]>,
    },
    /**
     * @zh 默认值
     * @en Default value
     */
    defaultValue: {
      type: Array as PropType<(Date | string | number)[]>,
    },
    /**
     * @zh 默认面板显示的日期
     * @en The date displayed in the default panel
     * */
    pickerValue: {
      type: Array as PropType<(Date | string | number)[]>,
    },
    /**
     * @zh 面板显示的日期
     * @en Date displayed on the panel
     * */
    defaultPickerValue: {
      type: Array as PropType<(Date | string | number)[]>,
    },
    /**
     * @zh 是否禁用
     * @en Whether to disable
     * */
    disabled: {
      type: [Boolean, Array] as PropType<boolean | boolean[]>,
      default: false,
    },
    /**
     * @zh 每周的第一天开始于周几，0 - 周日，1 - 周一。(默认0)
     * @en The first day of the week starts on the day of the week, 0-Sunday, 1-Monday. (Default 0)
     * @type number
     */
    dayStartOfWeek: {
      type: Number as PropType<0 | 1>,
      default: 0,
    },
    /**
     * @zh 展示日期的格式，参考[字符串解析格式](#字符串解析格式)
     * @en Display the format of the date, refer to [String Parsing Format](#String Parsing Format)
     * */
    format: {
      type: String,
    },
    /**
     * @zh 是否增加时间选择
     * @en Whether to increase time selection
     * */
    showTime: {
      type: Boolean,
    },
    /**
     * @zh 时间显示的参数，参考 [TimePickerProps](/vue/component/time-picker)
     * @en Time display parameters, refer to [TimePickerProps](/vue/component/time-picker)
     * */
    timePickerProps: {
      type: Object as PropType<Partial<TimePickerProps>>,
    },
    /**
     * @zh 提示文案
     * @en Prompt copy
     * */
    placeholder: {
      type: Array as PropType<string[]>,
    },
    /**
     * @zh 不可选的日期
     * @en Non-selectable date
     * */
    disabledDate: {
      type: Function as PropType<
        (current: Date, type: 'start' | 'end') => boolean
      >,
    },
    /**
     * @zh 不可选取的时间
     * @en Unselectable time
     * */
    disabledTime: {
      type: Function as PropType<
        (current: Date, type: 'start' | 'end') => DisabledTimeProps
      >,
    },
    /**
     * @zh 范围选择器输入框内的分割符号
     * @en The segmentation symbol in the input box of the range selector
     * */
    separator: {
      type: String,
    },
    popupContainer: {
      type: [String, Object] as PropType<
        string | HTMLElement | null | undefined
      >,
    },
    locale: {
      type: Object as PropType<Record<string, any>>,
    },
    hideTrigger: {
      type: Boolean,
    },
    allowClear: {
      type: Boolean,
      default: true,
    },
    readonly: {
      type: Boolean,
    },
    error: {
      type: Boolean,
    },
    size: {
      type: String as PropType<'mini' | 'small' | 'medium' | 'large'>,
      default: 'medium',
    },
    shortcuts: {
      type: Array as PropType<ShortcutType[]>,
      default: () => [],
    },
    shortcutsPosition: {
      type: String as PropType<'left' | 'bottom' | 'right'>,
      default: 'bottom',
    },
    position: {
      type: String as PropType<'top' | 'tl' | 'tr' | 'bottom' | 'bl' | 'br'>,
      default: 'bl',
    },
    popupVisible: {
      type: Boolean,
      default: undefined,
    },
    defaultPopupVisible: {
      type: Boolean,
    },
    triggerProps: {
      type: Object as PropType<Record<string, unknown>>,
    },
    unmountOnClose: {
      type: Boolean,
    },
  },
  emits: [
    /**
     * @zh 组件值发生改变
     * @en The component value changes
     * @param {(string | undefined)[] | undefined} dateString
     * @param {(Date | undefined)[] | undefined} date
     */
    'change',
    /**
     * @zh 选中日期发生改变但组件值未改变
     * @en The selected date has changed but the component value has not changed
     * @param {(string | undefined)[]} dateString
     * @param {(Date | undefined)[]} date
     */
    'select',
    'update:modelValue',
    /**
     * @zh 打开或关闭弹出框
     * @en Open or close the pop-up box
     * @param {boolean} visible
     */
    'popup-visible-change',
    'update:popupVisible',
    /**
     * @zh 点击确认按钮
     * @en Click the confirm button
     * @param {string[]} dateString
     * @param {Date[]} date
     */
    'ok',
    /**
     * @zh 点击清除按钮
     * @en Click the clear button
     */
    'clear',
    /**
     * @zh 点击快捷选项
     * @en Click on the shortcut option
     * @param {ShortcutType} shortcut
     */
    'select-shortcut',
    /**
     * @zh 面板日期改变
     * @en Panel date change
     * @param {string[]} dateString
     * @param {Date[]} date
     */
    'picker-value-change',
    'update:pickerValue',
  ],
  setup(props: RangePickerProps, { emit, slots }) {
    const {
      mode,
      showTime,
      format,
      modelValue,
      defaultValue,
      popupVisible,
      defaultPopupVisible,
      placeholder,
      timePickerProps,
      disabled,
      disabledDate,
      disabledTime,
      locale,
      pickerValue,
      defaultPickerValue,
    } = toRefs(props);

    const datePickerT = useProvideDatePickerTransform(
      reactive({
        locale,
      })
    );

    const prefixCls = getPrefixCls('picker');

    const computedPlaceholder = computed(
      () =>
        placeholder?.value ||
        ({
          date: datePickerT('datePicker.rangePlaceholder.date'),
          month: datePickerT('datePicker.rangePlaceholder.month'),
          year: datePickerT('datePicker.rangePlaceholder.year'),
          week: datePickerT('datePicker.rangePlaceholder.week'),
          quarter: datePickerT('datePicker.rangePlaceholder.quarter'),
        }[mode.value] as unknown as string[]) ||
        (datePickerT('datePicker.rangePlaceholder.date') as unknown as string[])
    );

    const computedFormat = useFormat(
      reactive({
        mode,
        format,
        showTime,
      })
    );

    const disabledArray = computed(() => {
      const disabled0 =
        disabled.value === true ||
        (isArray(disabled.value) && disabled.value[0] === true);
      const disabled1 =
        disabled.value === true ||
        (isArray(disabled.value) && disabled.value[1] === true);
      return [disabled0, disabled1];
    });

    const triggerDisabled = computed(
      () => disabledArray.value[0] && disabledArray.value[1]
    );

    function getFocusedIndex(cur = 0) {
      return disabledArray.value[cur] ? cur ^ 1 : cur;
    }

    const refInput = ref();
    const focusedIndex = ref(getFocusedIndex());
    const nextFocusedIndex = computed(() => {
      const cur = focusedIndex.value;
      const next = cur ^ 1;
      return disabledArray.value[next] ? cur : next;
    });
    const isNextDisabled = computed(
      () => disabledArray.value[focusedIndex.value ^ 1]
    );

    const { value: selectedValue, setValue: setSelectedValue } =
      useRangePickerState(
        reactive({
          modelValue,
          defaultValue,
          format: computedFormat,
        })
      );

    const [processValue, setProcessValue] = useState<
      Array<Dayjs | undefined> | undefined
    >();

    const [previewValue, setPreviewValue] = useState<
      Array<Dayjs | undefined> | undefined
    >();

    const panelValue = computed(
      () => previewValue.value || processValue.value || selectedValue.value
    );

    // input 操作的值
    const [inputValue, setInputValue] = useState<
      Array<string | undefined> | undefined
    >();

    const [panelVisible, setLocalPanelVisible] = useMergeState(
      defaultPopupVisible.value,
      reactive({ value: popupVisible })
    );
    const setPanelVisible = (newVisible: boolean) => {
      if (panelVisible.value !== newVisible) {
        setLocalPanelVisible(newVisible);
        emit('popup-visible-change', newVisible);
        emit('update:popupVisible', newVisible);
      }
    };

    // headerValue is used to generate the content displayed on the panel
    const {
      startHeaderValue,
      endHeaderValue,
      startHeaderOperations,
      endHeaderOperations,
      resetHeaderValue,
    } = useRangeHeaderValue(
      reactive({
        mode,
        value: pickerValue,
        defaultValue: defaultPickerValue,
        selectedValue: panelValue,
        format: computedFormat,
        onChange: (newVal: Dayjs[]) => {
          const formattedValue = getFormattedValue(
            newVal,
            computedFormat.value
          ) as string[];
          const dateValue = getDateValue(newVal);
          emit('picker-value-change', formattedValue, dateValue);
          emit('update:pickerValue', formattedValue);
        },
      })
    );

    const footerValue = ref([
      panelValue.value[0] || getNow(),
      panelValue.value[1] || getNow(),
    ]);
    watch(panelValue, () => {
      const [value0, value1] = panelValue.value;
      footerValue.value[0] = value0 || footerValue.value[0];
      footerValue.value[1] = value1 || footerValue.value[1];
    });

    const [timePickerValue, setTimePickerValue, resetTimePickerValue] =
      useRangeTimePickerValue(
        reactive({
          timePickerProps,
          selectedValue: panelValue,
        })
      );

    const isDateTime = computed(() => mode.value === 'date' && showTime.value);

    const isDisabledDate = useIsDisabledDate(
      reactive({
        mode,
        isRange: true,
        showTime,
        disabledDate,
        disabledTime,
      })
    );

    // needConfirm logic
    const needConfirm = computed(() => isDateTime.value);
    const confirmBtnDisabled = computed(
      () =>
        needConfirm.value &&
        (!isCompleteRangeValue(panelValue.value) ||
          isDisabledDate(panelValue.value[0], 'start') ||
          isDisabledDate(panelValue.value[1], 'end'))
    );

    watch(panelVisible, (newVisible) => {
      setProcessValue(undefined);
      setPreviewValue(undefined);
      // open
      if (newVisible) {
        resetHeaderValue();
        resetTimePickerValue();
        focusedIndex.value = getFocusedIndex(focusedIndex.value);
        nextTick(() => focusInput(focusedIndex.value));
      }
      // close
      if (!newVisible) {
        setInputValue(undefined);
      }
    });

    watch(focusedIndex, () => {
      focusInput(focusedIndex.value);
      setInputValue(undefined);
    });

    function emitChange(
      value: Array<Dayjs | undefined> | undefined,
      emitOk?: boolean
    ) {
      const formattedValue = getFormattedValue(value, computedFormat.value);
      const dateValue = getDateValue(value);
      if (isValueChange(value, selectedValue.value)) {
        emit('change', formattedValue, dateValue);
        emit('update:modelValue', formattedValue);
      }
      if (emitOk) {
        emit('ok', formattedValue, dateValue);
      }
    }

    function confirm(
      value: Array<Dayjs | undefined> | undefined,
      showPanel: boolean,
      emitOk?: boolean
    ) {
      if (
        isDisabledDate(value?.[0], 'start') ||
        isDisabledDate(value?.[1], 'end')
      ) {
        return;
      }

      let newValue = value ? [...value] : undefined;

      if (isCompleteRangeValue(newValue)) {
        newValue = getSortedDayjsArray(newValue);
      }

      emitChange(newValue, emitOk);
      setSelectedValue(newValue || []);
      setProcessValue(undefined);
      setPreviewValue(undefined);
      setInputValue(undefined);
      setPanelVisible(showPanel);
    }

    function select(
      value: Array<Dayjs | undefined>,
      options?: {
        emitSelect?: boolean;
        updateHeader?: boolean;
      }
    ) {
      const { emitSelect = false, updateHeader = false } = options || {};
      setProcessValue(value);
      setPreviewValue(undefined);
      setInputValue(undefined);

      if (emitSelect) {
        const formattedValue = getFormattedValue(value, computedFormat.value);
        const dateValue = getDateValue(value);
        emit('select', formattedValue, dateValue);
      }

      if (updateHeader) {
        resetHeaderValue();
      }
    }

    function preview(
      value: Array<Dayjs | undefined>,
      options?: {
        emitSelect?: boolean;
        updateHeader?: boolean;
      }
    ) {
      const { updateHeader = false } = options || {};
      setPreviewValue(value);
      setInputValue(undefined);

      if (updateHeader) {
        resetHeaderValue();
      }
    }

    function focusInput(index?: number) {
      refInput.value && refInput.value.focus && refInput.value.focus(index);
    }

    function getMergedOpValue(date: Dayjs, time?: Dayjs) {
      if (!isDateTime.value) return date;
      return mergeValueWithTime(getNow(), date, time);
    }

    function onPanelVisibleChange(visible: boolean) {
      setPanelVisible(visible);
    }

    function onPanelCellMouseEnter(date: Dayjs) {
      if (
        processValue.value &&
        panelValue.value[nextFocusedIndex.value] &&
        (!needConfirm.value || !isCompleteRangeValue(processValue.value))
      ) {
        const newValue = [...panelValue.value];
        const mergedOpValue = getMergedOpValue(
          date,
          timePickerValue.value[focusedIndex.value]
        );
        newValue[focusedIndex.value] = mergedOpValue;

        preview(newValue);
      }
    }

    function getValueToModify(isTime = false) {
      if (isNextDisabled.value) return [...selectedValue.value];
      if (processValue.value) {
        return isTime || !isCompleteRangeValue(processValue.value)
          ? [...processValue.value]
          : [];
      }
      return isTime ? [...selectedValue.value] : [];
    }

    function onPanelCellClick(date: Dayjs) {
      const newValue = getValueToModify();
      const mergedOpValue = getMergedOpValue(
        date,
        timePickerValue.value[focusedIndex.value]
      );
      newValue[focusedIndex.value] = mergedOpValue;

      if (!needConfirm.value && isCompleteRangeValue(newValue)) {
        confirm(newValue, false);
      } else {
        select(newValue, { emitSelect: true });
        if (!isCompleteRangeValue(newValue)) {
          focusedIndex.value = nextFocusedIndex.value;
        }
      }
    }

    function onTimePickerSelect(time: Dayjs, type: 'start' | 'end') {
      const updateIndex = type === 'start' ? 0 : 1;
      const mergedOpValue = getMergedOpValue(
        timePickerValue.value[updateIndex],
        time
      );
      const newTimeValue = [...timePickerValue.value];
      newTimeValue[updateIndex] = mergedOpValue;
      setTimePickerValue(newTimeValue);

      const newValue = getValueToModify(true);
      if (newValue[updateIndex]) {
        newValue[updateIndex] = mergedOpValue;
        select(newValue, { emitSelect: true });
      }
    }

    function onPanelShortcutMouseEnter(value: Array<Dayjs | undefined>) {
      preview(value, { updateHeader: true });
      resetHeaderValue();
    }

    function onPanelShortcutMouseLeave() {
      setProcessValue(undefined);
      setPreviewValue(undefined);
      setInputValue(undefined);
      resetHeaderValue();
    }

    function onPanelShortcutClick(
      value: Array<Dayjs | undefined>,
      shortcut: ShortcutType
    ) {
      emit('select-shortcut', shortcut);
      confirm(value, false);
    }

    function onPanelConfirm() {
      confirm(panelValue.value, false, true);
    }

    function onInputClear() {
      confirm(undefined, true);
      emit('clear');
    }

    function onInputChange(e: any) {
      setPanelVisible(true);

      const targetValue = e.target.value;

      // TODO: Null value should be restored to the current value, invalid when deleted as a whole
      if (!targetValue) {
        setInputValue(undefined);
        return;
      }

      const formattedPanelValue = getFormattedValue(
        panelValue.value,
        computedFormat.value
      ) as Array<string | undefined>;

      const newInputValue = isArray(inputValue.value)
        ? [...inputValue.value]
        : formattedPanelValue || [];

      newInputValue[focusedIndex.value] = targetValue;

      setInputValue(newInputValue);

      if (!isValidInputValue(targetValue, computedFormat.value)) return;

      const targetValueDayjs = dayjs(targetValue, computedFormat.value);

      if (
        isDisabledDate(
          targetValueDayjs,
          focusedIndex.value === 0 ? 'start' : 'end'
        )
      )
        return;

      const newValue = isArray(panelValue.value) ? [...panelValue.value] : [];
      newValue[focusedIndex.value] = targetValueDayjs;

      select(newValue, { updateHeader: true });
      resetHeaderValue();
    }

    function onInputPressEnter() {
      if (isValidRangeValue(panelValue.value)) {
        confirm(panelValue.value, false);
      } else {
        focusedIndex.value = nextFocusedIndex.value;
      }
    }

    const computedTimePickerProps = computed(() => ({
      format: computedFormat.value,
      ...omit(timePickerProps?.value || {}, ['defaultValue']),
      visible: panelVisible.value,
    }));

    const headerIcons = computed(() => ({
      prev: slots['icon-prev'],
      prevDouble: slots['icon-prev-double'],
      next: slots['icon-next'],
      nextDouble: slots['icon-next-double'],
    }));

    const startHeaderProps = reactive({
      headerValue: startHeaderValue,
      headerOperations: startHeaderOperations,
      headerIcons,
    });

    const endHeaderProps = reactive({
      headerValue: endHeaderValue,
      headerOperations: endHeaderOperations,
      headerIcons,
    });

    const rangePanelProps = computed(() => ({
      ...pick(props as Record<keyof RangePickerProps, any>, [
        'mode',
        'showTime',
        'shortcuts',
        'shortcutsPosition',
        'dayStartOfWeek',
        'disabledDate',
        'disabledTime',
        'hideTrigger',
      ]),
      prefixCls,
      format: computedFormat.value,
      value: panelValue.value,
      showConfirmBtn: needConfirm.value,
      confirmBtnDisabled: confirmBtnDisabled.value,
      timePickerValue: timePickerValue.value,
      timePickerProps: computedTimePickerProps.value,
      extra: slots.extra,
      dateRender: slots.cell,
      startHeaderProps,
      endHeaderProps,
      footerValue: footerValue.value,
      disabled: disabledArray.value,
      visible: panelVisible.value,
      onCellClick: onPanelCellClick,
      onCellMouseEnter: onPanelCellMouseEnter,
      onShortcutClick: onPanelShortcutClick,
      onShortcutMouseEnter: onPanelShortcutMouseEnter,
      onShortcutMouseLeave: onPanelShortcutMouseLeave,
      onConfirm: onPanelConfirm,
      onTimePickerSelect,
    }));

    return {
      prefixCls,
      refInput,
      computedFormat,
      computedPlaceholder,
      panelVisible,
      panelValue,
      inputValue,
      focusedIndex,
      triggerDisabled,
      onPanelVisibleChange,
      onInputClear,
      onInputChange,
      onInputPressEnter,
      rangePanelProps,
    };
  },
});
</script>
