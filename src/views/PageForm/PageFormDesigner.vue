<template>
  <section class="page-form-designer">
    <header class="designer-header">
      <div>
        <h2>表单设计器</h2>
        <p>从左侧添加控件，在画布中选择控件后配置属性。导出的结构可直接作为 <code>PageForm</code> 的 <code>tempForm</code> 使用。</p>
      </div>
      <div class="designer-actions">
        <el-button @click="previewVisible = true">预览</el-button>
        <el-button @click="showJson">导出 JSON</el-button>
        <el-button type="danger" plain @click="resetDesigner">重置</el-button>
      </div>
    </header>

    <main class="designer-body">
      <aside class="designer-panel component-panel">
        <h3>组件库</h3>
        <p class="panel-tip">选择分组后，新增字段会放入该分组。</p>
        <el-divider content-position="left">基础字段</el-divider>
        <el-button v-for="item in fieldPalette" :key="item.formType" class="palette-button" @click="addField(item)">
          {{ item.label }}
        </el-button>
        <el-divider content-position="left">布局</el-divider>
        <el-button class="palette-button" @click="addGroup">分组</el-button>
        <el-button class="palette-button" @click="addTable">表格分组</el-button>
      </aside>

      <section class="canvas-panel" @click.self="selectedId = ''">
        <div class="canvas-title">
          <strong>表单画布</strong>
          <el-radio-group v-model="form.cols" size="small">
            <el-radio-button :value="1">1 列</el-radio-button>
            <el-radio-button :value="2">2 列</el-radio-button>
            <el-radio-button :value="3">3 列</el-radio-button>
          </el-radio-group>
        </div>
        <div v-if="form.form.length === 0" class="empty-canvas">请从左侧添加字段或布局组件。</div>
        <div v-else class="canvas-items">
          <template v-for="item in form.form" :key="item.ident">
            <div v-if="isContainer(item)" class="canvas-group" :class="{ selected: selectedId === item.ident }" @click.stop="select(item)">
              <div class="canvas-group-title">
                <span>{{ item.label || '未命名分组' }}</span>
                <small>{{ item.formType === PageFormGroup.Table ? '表格分组' : '分组' }}</small>
              </div>
              <div class="canvas-group-fields">
                <template v-for="child in containerFields(item)" :key="child.ident">
                  <FieldCard :item="child" :selected="selectedId === child.ident" @select="select" />
                </template>
                <span v-if="containerFields(item).length === 0" class="empty-group">请选择此分组后添加字段</span>
              </div>
            </div>
            <FieldCard v-else :item="item" :selected="selectedId === item.ident" @select="select" />
          </template>
        </div>
      </section>

      <aside class="designer-panel property-panel">
        <h3>属性配置</h3>
        <template v-if="selectedItem">
          <el-form label-position="top" size="default">
            <el-form-item label="组件标题"><el-input v-model="selectedItem.label" placeholder="请输入标题" /></el-form-item>
            <el-form-item v-if="!isContainer(selectedItem)" label="字段名">
              <el-input v-model="selectedItem.fieldName" placeholder="例如 name" />
            </el-form-item>
            <el-form-item v-if="!isContainer(selectedItem)" label="占位提示">
              <el-input v-model="selectedItem.placeholder" placeholder="为空时自动生成" />
            </el-form-item>
            <el-form-item v-if="!isContainer(selectedItem)" label="必填"><el-switch v-model="selectedItem.must" /></el-form-item>
            <el-form-item v-if="isChoice(selectedItem)" label="选项（每行：标签,值）">
              <el-input v-model="selectedItem.optionText" type="textarea" :rows="5" placeholder="启用,1&#10;停用,0" @change="syncOptions(selectedItem)" />
            </el-form-item>
          </el-form>
          <div class="property-actions">
            <el-button :disabled="!canMove(-1)" @click="moveSelected(-1)">上移</el-button>
            <el-button :disabled="!canMove(1)" @click="moveSelected(1)">下移</el-button>
            <el-button type="danger" @click="removeSelected">删除</el-button>
          </div>
        </template>
        <el-empty v-else description="请选择画布中的组件" :image-size="88" />
      </aside>
    </main>

    <el-dialog v-model="previewVisible" title="表单预览" width="860px" destroy-on-close>
      <PageForm :temp-form="previewForm" />
    </el-dialog>
    <el-dialog v-model="jsonVisible" title="PageForm 配置" width="860px">
      <el-input v-model="jsonText" type="textarea" :rows="26" readonly />
    </el-dialog>
  </section>
</template>

<script setup lang="ts">
import { computed, defineComponent, h, reactive, ref, type PropType } from 'vue'
import type { FormItemRule } from 'element-plus'
import PageForm from '@/components/PageForm/PageForm.vue'
import { PageFormGroup, PageFormType, type TempEditForm, type TempEditPageData } from '@/components/PageForm'

type DesignerItem = Omit<TempEditForm, 'child' | 'ident'> & { ident: string; child?: DesignerItem[]; must?: boolean; optionText?: string }
type PaletteItem = { label: string; formType: number; choice?: boolean }

const fieldPalette: PaletteItem[] = [
  { label: '单行文本', formType: PageFormType.Input }, { label: '多行文本', formType: PageFormType.TextAreaInput },
  { label: '数字', formType: PageFormType.Number }, { label: '开关', formType: PageFormType.Switch },
  { label: '日期', formType: PageFormType.DateSelect }, { label: '日期时间', formType: PageFormType.DateTimeSelect },
  { label: '下拉选择', formType: PageFormType.OneSelectSearch, choice: true }, { label: '单选', formType: PageFormType.Radio, choice: true },
  { label: '多选', formType: PageFormType.MultSelect, choice: true },
]

const FieldCard = defineComponent({
  props: { item: { type: Object as PropType<DesignerItem>, required: true }, selected: Boolean },
  emits: ['select'],
  setup(props, { emit }) {
    return () => h('div', { class: ['field-card', { selected: props.selected }], onClick: (event: Event) => { event.stopPropagation(); emit('select', props.item) } }, [
      h('strong', props.item.label || '未命名字段'), h('span', props.item.fieldName || '请配置字段名'),
    ])
  },
})

const form = reactive({ cols: 2, form: [] as DesignerItem[] })
const selectedId = ref('')
const previewVisible = ref(false)
const jsonVisible = ref(false)
const jsonText = ref('')
let id = 0
const nextId = () => `page-form-${Date.now()}-${++id}`
const isContainer = (item: DesignerItem) => [PageFormGroup.Group, PageFormGroup.Table, PageFormGroup.List].includes(item.formType)
const isChoice = (item: DesignerItem) => [PageFormType.OneSelect, PageFormType.OneSelectSearch, PageFormType.MultSelect, PageFormType.Radio, PageFormType.Checkbox].includes(item.formType)
const containerFields = (item: DesignerItem): DesignerItem[] => item.formType === PageFormGroup.Table || item.formType === PageFormGroup.List ? ((item.child?.[0]?.child ?? []) as DesignerItem[]) : ((item.child ?? []) as DesignerItem[])

const walk = (items: DesignerItem[], target: string): DesignerItem | undefined => {
  for (const item of items) { if (item.ident === target) return item; const found = walk((item.child ?? []) as DesignerItem[], target); if (found) return found }
}
const selectedItem = computed(() => selectedId.value ? walk(form.form, selectedId.value) : undefined)
const select = (item: DesignerItem) => { selectedId.value = item.ident }
const createField = (palette: PaletteItem): DesignerItem => {
  const base = palette.label.replace(/[^a-zA-Z]/g, '').toLowerCase() || 'field'
  return { ident: nextId(), formType: palette.formType, label: palette.label, fieldName: `${base}${id}`, must: false, optionText: palette.choice ? '选项一,option1\n选项二,option2' : '' }
}
const addField = (palette: PaletteItem) => {
  const item = createField(palette)
  const target = selectedItem.value
  if (target && isContainer(target)) containerFields(target).push(item)
  else form.form.push(item as never)
  select(item)
}
const addGroup = () => { const item: DesignerItem = { ident: nextId(), formType: PageFormGroup.Group, label: '基础信息', fieldName: '', child: [] }; form.form.push(item as never); select(item) }
const addTable = () => {
  const item: DesignerItem = { ident: nextId(), formType: PageFormGroup.Table, label: '明细信息', fieldName: `details${id}`, child: [{ ident: nextId(), formType: PageFormGroup.Group, label: '', fieldName: '', child: [] }] }
  form.form.push(item as never); select(item)
}
const syncOptions = (item: DesignerItem) => { item.optionText = item.optionText ?? '' }
const parentAndIndex = (items: DesignerItem[], target: string): { items: DesignerItem[]; index: number } | undefined => {
  const index = items.findIndex((item) => item.ident === target)
  if (index >= 0) return { items, index }
  for (const item of items) { const found = parentAndIndex((item.child ?? []) as DesignerItem[], target); if (found) return found }
}
const canMove = (offset: number) => { const result = parentAndIndex(form.form, selectedId.value); return !!result && result.index + offset >= 0 && result.index + offset < result.items.length }
const moveSelected = (offset: number) => { const result = parentAndIndex(form.form, selectedId.value); if (result && canMove(offset)) [result.items[result.index], result.items[result.index + offset]] = [result.items[result.index + offset], result.items[result.index]] }
const removeSelected = () => { const result = parentAndIndex(form.form, selectedId.value); if (result) result.items.splice(result.index, 1); selectedId.value = '' }

const optionsFromItem = (item: DesignerItem) => (item.optionText ?? '').split('\n').map((line) => line.trim()).filter(Boolean).map((line) => {
  const [label, ...rest] = line.split(','); return { label: label.trim(), value: (rest.join(',').trim() || label.trim()) }
})
const cloneItem = (item: DesignerItem): TempEditForm => ({ formType: item.formType, label: item.label, fieldName: item.fieldName, placeholder: item.placeholder || undefined, child: item.child?.map(cloneItem) })
const buildOptions = (items: DesignerItem[], result: TempEditPageData['options'] = {}) => { items.forEach((item) => { if (isChoice(item) && item.fieldName) result[item.fieldName] = optionsFromItem(item); if (item.child) buildOptions(item.child as DesignerItem[], result) }); return result }
const buildData = (items: DesignerItem[], result: Record<string, unknown> = {}) => { items.forEach((item) => { if (item.formType === PageFormGroup.Table || item.formType === PageFormGroup.List) { if (item.fieldName) result[item.fieldName] = [] } else if (!isContainer(item) && item.fieldName) result[item.fieldName] = item.formType === PageFormType.MultSelect || item.formType === PageFormType.Checkbox ? [] : item.formType === PageFormType.Switch ? false : ''; if (item.formType === PageFormGroup.Group) buildData((item.child ?? []) as DesignerItem[], result) }); return result }
const buildRules = (items: DesignerItem[], result: Record<string, FormItemRule[] | Record<string, FormItemRule[]>> = {}) => { items.forEach((item) => { if (item.formType === PageFormGroup.Table || item.formType === PageFormGroup.List) { const nested: Record<string, FormItemRule[]> = {}; buildRules(containerFields(item), nested); if (item.fieldName && Object.keys(nested).length) result[item.fieldName] = nested } else if (item.formType === PageFormGroup.Group) buildRules((item.child ?? []) as DesignerItem[], result); else if (item.must && item.fieldName) result[item.fieldName] = [{ required: true, message: `请${isChoice(item) ? '选择' : '输入'}${item.label}`, trigger: isChoice(item) ? 'change' : 'blur' }] }); return result }
const previewForm = computed<TempEditPageData>(() => ({ loading: false, hideBtn: true, cols: form.cols, form: form.form.map(cloneItem), formData: buildData(form.form), options: buildOptions(form.form), rules: buildRules(form.form) }))
const exportData = () => ({ ...previewForm.value, formData: buildData(form.form), options: buildOptions(form.form), rules: buildRules(form.form) })
const showJson = () => { jsonText.value = JSON.stringify(exportData(), null, 2); jsonVisible.value = true }
const resetDesigner = () => { form.form.splice(0); selectedId.value = '' }
</script>

<style scoped lang="less">
.page-form-designer { min-height: 720px; background: #f5f7fa; }
.designer-header { display: flex; align-items: center; justify-content: space-between; padding: 18px 24px; background: #fff; border-bottom: 1px solid #ebeef5; h2 { margin: 0 0 6px; font-size: 20px; } p { margin: 0; color: #909399; font-size: 13px; } }
.designer-actions, .property-actions { display: flex; gap: 8px; }
.designer-body { display: grid; grid-template-columns: 200px minmax(420px, 1fr) 280px; min-height: 650px; }
.designer-panel { padding: 18px; background: #fff; border-right: 1px solid #ebeef5; h3 { margin: 0 0 14px; font-size: 16px; } }
.property-panel { border-right: 0; border-left: 1px solid #ebeef5; }.panel-tip { margin: 0; color: #909399; font-size: 12px; line-height: 1.5; }
.palette-button { width: 100%; margin: 0 0 8px !important; text-align: left; }.canvas-panel { padding: 20px; overflow: auto; }.canvas-title { display: flex; justify-content: space-between; align-items: center; margin-bottom: 16px; }.empty-canvas, .empty-group { display: block; padding: 42px 16px; color: #909399; text-align: center; border: 1px dashed #cdd0d6; background: #fff; }.canvas-items { display: grid; gap: 12px; }.canvas-group { padding: 16px; border: 1px solid #dcdfe6; background: #fff; cursor: pointer; }.canvas-group.selected, .field-card.selected { border-color: #409eff; box-shadow: 0 0 0 2px rgb(64 158 255 / 15%); }.canvas-group-title { display: flex; justify-content: space-between; padding-bottom: 10px; font-weight: 600; small { color: #909399; font-weight: normal; } }.canvas-group-fields { display: grid; grid-template-columns: repeat(2, minmax(0, 1fr)); gap: 10px; }.field-card { padding: 12px; border: 1px solid #ebeef5; background: #fff; cursor: pointer; strong, span { display: block; } span { margin-top: 5px; color: #909399; font-size: 12px; }.property-actions { padding-top: 10px; flex-wrap: wrap; }
@media (max-width: 1050px) { .designer-body { grid-template-columns: 180px minmax(360px, 1fr); }.property-panel { grid-column: 1 / -1; border-top: 1px solid #ebeef5; border-left: 0; }.designer-header { align-items: flex-start; gap: 12px; flex-direction: column; } }
</style>
