<template>
  <div class="page-form-designer">
    <div class="designer-header">
      <div>
        <h2>表单设计器</h2>
        <span>拖拽左侧控件到中间画布，选中控件后可在右侧配置属性</span>
      </div>
      <div class="designer-actions">
        <el-button @click="previewVisible = true">预览</el-button>
        <el-button @click="exportJson">导出 JSON</el-button>
        <el-button @click="resetDesigner">重置</el-button>
      </div>
    </div>

    <div class="designer-body">
      <YzCustomForm
        ref="designerRef"
        :form="formData.form"
        :attr-data="formData.attrData"
        :component-group-list="componentGroupList"
        @release="handleRelease"
      />
    </div>

    <el-dialog v-model="previewVisible" title="表单预览" width="760px" destroy-on-close>
      <PageForm :temp-form="previewForm" />
    </el-dialog>

    <el-dialog v-model="jsonVisible" title="表单 JSON" width="800px">
      <el-input v-model="jsonText" type="textarea" :rows="24" readonly />
    </el-dialog>
  </div>
</template>

<script setup lang="ts">
import { computed, ref } from 'vue'
import PageForm from '@/components/PageForm/PageForm.vue'
import { PageFormGroup, PageFormType } from '@/components/PageForm'
import type { TempEditPageData } from '@/components/PageForm'
import YzCustomForm from '@/components/common/YzCustomForm/index.vue'
import { defaultAttr, defaultGroupAttr, defaultSelectAttr } from '@/components/common/YzCustomForm/index'
import type {
  componentGroup,
  componentItem,
  componentForm,
  ReleaseData,
  matterExpose,
  formItemOption,
} from '@/components/common/YzCustomForm/index'

type DesignerData = ReleaseData

const createOptions = (items: Array<[string, string]>): formItemOption[] => items.map(([label, value]) => ({
  title: label,
  label,
  value,
  disabled: false,
  isDefault: false,
  child: [],
}))

const cloneAttrs = (attrs: any[]) => attrs.map((item) => ({ ...item }))

const componentGroupList: componentGroup[] = [
  {
    label: '基础控件',
    code: 'basic',
    children: [
      {
        label: '单行文本',
        icon: '/imgs/matter/amount.png',
        type: PageFormType.Input,
        code: 'input',
        isIndependent: true,
        isPreview: true,
        isAddGroup: false,
        isButton: false,
        attr: cloneAttrs(defaultAttr),
      },
      {
        label: '多行文本',
        icon: '/imgs/matter/amount.png',
        type: PageFormType.TextAreaInput,
        code: 'textarea',
        isIndependent: true,
        isPreview: true,
        isAddGroup: false,
        isButton: false,
        attr: cloneAttrs(defaultAttr),
      },
      {
        label: '数字',
        icon: '/imgs/matter/amount.png',
        type: PageFormType.Number,
        code: 'number',
        isIndependent: true,
        isPreview: true,
        isAddGroup: false,
        isButton: false,
        attr: cloneAttrs(defaultAttr),
      },
      {
        label: '开关',
        icon: '/imgs/matter/amount.png',
        type: PageFormType.Switch,
        code: 'switch',
        isIndependent: true,
        isPreview: true,
        isAddGroup: false,
        isButton: false,
        attr: cloneAttrs(defaultAttr),
      },
      {
        label: '日期',
        icon: '/imgs/matter/amount.png',
        type: PageFormType.DateSelect,
        code: 'date',
        isIndependent: true,
        isPreview: true,
        isAddGroup: false,
        isButton: false,
        attr: cloneAttrs(defaultAttr),
      },
      {
        label: '日期时间',
        icon: '/imgs/matter/amount.png',
        type: PageFormType.TimeSelect,
        code: 'datetime',
        isIndependent: true,
        isPreview: true,
        isAddGroup: false,
        isButton: false,
        attr: cloneAttrs(defaultAttr),
      },
    ],
  },
  {
    label: '选择控件',
    code: 'select',
    children: [
      {
        label: '下拉选择',
        icon: '/imgs/matter/amount.png',
        type: PageFormType.OneSelectSearch,
        code: 'select',
        isIndependent: true,
        isPreview: true,
        isAddGroup: false,
        isButton: false,
        attr: cloneAttrs(defaultSelectAttr).map((item) => item.code === 'dataSource'
          ? { ...item, data: createOptions([['选项一', 'option1'], ['选项二', 'option2']]) }
          : item),
      },
      {
        label: '单选',
        icon: '/imgs/matter/amount.png',
        type: PageFormType.Radio,
        code: 'radio',
        isIndependent: true,
        isPreview: true,
        isAddGroup: false,
        isButton: false,
        attr: cloneAttrs(defaultSelectAttr).map((item) => item.code === 'dataSource'
          ? { ...item, data: createOptions([['是', 'yes'], ['否', 'no']]) }
          : item),
      },
      {
        label: '多选',
        icon: '/imgs/matter/amount.png',
        type: PageFormType.MultSelect,
        code: 'multiSelect',
        isIndependent: true,
        isPreview: true,
        isAddGroup: false,
        isButton: false,
        attr: cloneAttrs(defaultSelectAttr),
      },
    ],
  },
  {
    label: '布局控件',
    code: 'layout',
    children: [
      {
        label: '分组块',
        icon: '/imgs/matter/amount.png',
        type: PageFormGroup.Group,
        code: 'group',
        isIndependent: true,
        isPreview: true,
        isAddGroup: false,
        isButton: false,
        attr: cloneAttrs(defaultGroupAttr),
      },
      {
        label: '表格分组',
        icon: '/imgs/matter/amount.png',
        type: PageFormGroup.Table,
        code: 'table',
        isIndependent: true,
        isPreview: true,
        isAddGroup: false,
        isButton: false,
        attr: cloneAttrs(defaultGroupAttr),
      },
    ],
  },
]

const designerRef = ref<matterExpose>()
const previewVisible = ref(false)
const jsonVisible = ref(false)
const jsonText = ref('')
const formData = ref<DesignerData>({ form: [], attrData: {} })

const cloneForm = (items: componentForm[]): componentForm[] => items.map((item) => ({
  ...item,
  child: item.child ? cloneForm(item.child) : undefined,
}))

const getDefaultValue = (item: componentForm): unknown => {
  if (item.formType === PageFormType.Switch) return false
  if (item.formType === PageFormGroup.List) return []
  return (item as componentForm & { defaultValue?: unknown }).defaultValue ?? ''
}

const getFormData = (items: componentForm[], result: Record<string, unknown> = {}) => {
  items.forEach((item) => {
    if (item.fieldName) result[item.fieldName] = getDefaultValue(item)
    if (item.child) getFormData(item.child, result)
  })
  return result
}

const previewForm = computed<TempEditPageData>(() => ({
  design: false,
  form: cloneForm(formData.value.form),
  formData: getFormData(formData.value.form),
  options: {},
  hideBtn: true,
  cols: 1,
  loading: false,
}))

const handleRelease = (data: ReleaseData) => {
  formData.value = {
    form: cloneForm(data.form),
    attrData: JSON.parse(JSON.stringify(data.attrData ?? {})),
  }
}

const resetDesigner = () => {
  formData.value = { form: [], attrData: {} }
  jsonText.value = ''
}

const exportJson = () => {
  const data = designerRef.value?.getData() ?? formData.value
  jsonText.value = JSON.stringify(data, null, 2)
  jsonVisible.value = true
}
</script>

<style scoped lang="less">
.page-form-designer {
  display: flex;
  flex-direction: column;
  width: 100%;
  height: 100%;
  min-height: 720px;
  background: #f5f7fa;
}

.designer-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 18px 24px;
  background: #fff;
  border-bottom: 1px solid #ebeef5;

  h2 {
    margin: 0 0 6px;
    color: #303133;
    font-size: 20px;
  }

  span {
    color: #909399;
    font-size: 13px;
  }
}

.designer-actions {
  display: flex;
  gap: 8px;
}

.designer-body {
  flex: 1;
  min-height: 0;
  overflow: hidden;
}
</style>
