<template>
  <div class="attribute-page">
    <!-- 顶部工具条 -->
    <el-card class="toolbar-card" shadow="never">
      <div class="toolbar">
        <el-form :inline="true" :model="filters" class="toolbar-form">
          <el-form-item label="搜索">
            <el-input
              v-model="filters.search"
              placeholder="按属性名称/键名搜索"
              clearable
              @clear="handleFilterChange"
              @input="handleFilterChange"
            />
          </el-form-item>
          <el-form-item label="适用层级">
            <el-select
              v-model="filters.levels"
              multiple
              placeholder="选择层级"
              clearable
              collapse-tags
              @change="handleFilterChange"
            >
              <el-option
                v-for="item in levelOptions"
                :key="item.value"
                :label="item.label"
                :value="item.value"
              />
            </el-select>
          </el-form-item>
          <el-form-item label="继承策略">
            <el-select
              v-model="filters.strategies"
              multiple
              placeholder="选择策略"
              clearable
              collapse-tags
              @change="handleFilterChange"
            >
              <el-option
                v-for="item in inheritStrategyOptions"
                :key="item.value"
                :label="item.label"
                :value="item.value"
              />
            </el-select>
          </el-form-item>
          <el-form-item label="值类型">
            <el-select
              v-model="filters.valueTypes"
              multiple
              placeholder="选择类型"
              clearable
              collapse-tags
              @change="handleFilterChange"
            >
              <el-option
                v-for="item in valueTypeOptions"
                :key="item.value"
                :label="item.label"
                :value="item.value"
              />
            </el-select>
          </el-form-item>
          <el-form-item label="状态">
            <el-select
              v-model="filters.statuses"
              multiple
              placeholder="选择状态"
              clearable
              collapse-tags
              @change="handleFilterChange"
            >
              <el-option
                v-for="item in statusOptions"
                :key="item.value"
                :label="item.label"
                :value="item.value"
              />
            </el-select>
          </el-form-item>
        </el-form>
        <el-button type="primary" @click="openCreate">+ 新建属性</el-button>
      </div>
    </el-card>

    <!-- 主表格 -->
    <el-card class="table-card" shadow="never">
      <el-table
        :data="pagedAttributes"
        border
        row-key="id"
        class="attribute-table"
        :empty-text="emptyText"
      >
        <el-table-column type="expand" width="48">
          <template #default="scope">
            <div v-if="scope.row.value_type === 'enum'">
              <div class="enum-table-title">枚举值集</div>
              <el-table :data="scope.row.enums" size="small" border>
                <el-table-column prop="value" label="枚举值" min-width="120" />
                <el-table-column prop="label" label="说明" min-width="160" />
                <el-table-column label="状态" width="80">
                  <template #default="enumScope">
                    <el-tag
                      :type="enumScope.row.status === 'enabled' ? 'success' : 'info'"
                      effect="light"
                    >
                      {{ enumScope.row.status === 'enabled' ? '在用' : '停用' }}
                    </el-tag>
                  </template>
                </el-table-column>
              </el-table>
            </div>
            <div v-else class="enum-empty">无枚举值数据</div>
          </template>
        </el-table-column>
        <el-table-column label="属性名称" min-width="160">
          <template #default="scope">
            <el-link type="primary" @click="openView(scope.row)">{{ scope.row.name }}</el-link>
          </template>
        </el-table-column>
        <el-table-column prop="key" label="属性键名" min-width="140">
          <template #default="scope">
            <span class="monospace">{{ scope.row.key }}</span>
          </template>
        </el-table-column>
        <el-table-column label="适用层级" min-width="160">
          <template #default="scope">
            <el-tag
              v-for="level in scope.row.levels"
              :key="level"
              class="mr-4"
              type="info"
            >
              {{ levelLabelMap[level] || level }}
            </el-tag>
          </template>
        </el-table-column>
        <el-table-column label="继承策略" min-width="140">
          <template #default="scope">
            <el-tag :type="inheritTagTypeMap[scope.row.inherit_strategy]" effect="light">
              {{ inheritStrategyLabelMap[scope.row.inherit_strategy] || scope.row.inherit_strategy }}
            </el-tag>
          </template>
        </el-table-column>
        <el-table-column label="值类型" min-width="120">
          <template #default="scope">
            <el-tag :type="valueTypeTagMap[scope.row.value_type]" effect="light">
              {{ valueTypeLabelMap[scope.row.value_type] || scope.row.value_type }}
            </el-tag>
          </template>
        </el-table-column>
        <el-table-column label="状态" width="100" align="center">
          <template #default="scope">
            <el-tag :type="scope.row.status === 'enabled' ? 'success' : 'info'" effect="light">
              {{ scope.row.status === 'enabled' ? '在用' : '停用' }}
            </el-tag>
          </template>
        </el-table-column>
        <el-table-column prop="usage_count" label="使用数" width="90" align="center" />
        <el-table-column label="操作" width="150" fixed="right" align="center">
          <template #default="scope">
            <el-button type="primary" link size="small" @click="openEdit(scope.row)">编辑</el-button>
            <el-button
              :type="scope.row.status === 'enabled' ? 'warning' : 'success'"
              link
              size="small"
              @click="handleToggleStatus(scope.row)"
            >
              {{ scope.row.status === 'enabled' ? '停用' : '启用' }}
            </el-button>
          </template>
        </el-table-column>
      </el-table>
      <div class="pagination-wrap">
        <el-pagination
          v-model:current-page="pagination.currentPage"
          v-model:page-size="pagination.pageSize"
          :page-sizes="[10, 20, 50]"
          layout="total, sizes, prev, pager, next, jumper"
          :total="filteredAttributes.length"
        />
      </div>
    </el-card>

    <!-- 右侧抽屉 -->
    <el-drawer
      v-model="drawerVisible"
      :title="drawerTitle"
      size="520px"
      :close-on-click-modal="false"
      destroy-on-close
    >
      <el-form ref="drawerFormRef" :model="formModel" label-width="110px" class="drawer-form">
        <el-form-item label="属性名称" :required="true">
          <el-input
            v-model="formModel.name"
            :disabled="isReadOnly"
            placeholder="请输入属性名称"
          />
        </el-form-item>
        <el-form-item label="属性键名" :required="true" :error="keyError">
          <el-input
            v-model="formModel.key"
            :disabled="isReadOnly"
            placeholder="仅小写字母、数字、下划线"
          />
        </el-form-item>
        <el-form-item label="值类型">
          <el-select v-model="formModel.value_type" :disabled="isReadOnly">
            <el-option
              v-for="item in valueTypeOptions"
              :key="item.value"
              :label="item.label"
              :value="item.value"
            />
          </el-select>
        </el-form-item>
        <el-form-item label="适用层级" :required="true">
          <el-select
            v-model="formModel.levels"
            multiple
            :disabled="isReadOnly"
            placeholder="请选择至少一项"
          >
            <el-option
              v-for="item in levelOptions"
              :key="item.value"
              :label="item.label"
              :value="item.value"
            />
          </el-select>
        </el-form-item>
        <el-form-item label="继承策略">
          <el-select v-model="formModel.inherit_strategy" :disabled="isReadOnly">
            <el-option
              v-for="item in inheritStrategyOptions"
              :key="item.value"
              :label="item.label"
              :value="item.value"
            />
          </el-select>
        </el-form-item>
        <el-form-item v-if="formModel.inherit_strategy === 'equal'" label-width="0">
          <el-alert
            title="等于父级时默认值可留空，将跟随上级配置"
            type="info"
            :closable="false"
          />
        </el-form-item>
        <el-form-item label="默认值">
          <template v-if="formModel.value_type === 'enum'">
            <el-select
              v-model="formModel.default_value"
              :disabled="isReadOnly || usableEnumOptions.length === 0"
              placeholder="请选择默认枚举值"
              clearable
            >
              <el-option
                v-for="item in usableEnumOptions"
                :key="item.value"
                :label="item.label"
                :value="item.value"
              />
            </el-select>
          </template>
          <template v-else-if="formModel.value_type === 'bool'">
            <el-switch
              v-model="formModel.default_value"
              :disabled="isReadOnly"
              active-text="是"
              inactive-text="否"
            />
          </template>
          <template v-else-if="formModel.value_type === 'number'">
            <el-input-number
              v-model="formModel.default_value"
              :disabled="isReadOnly"
              :controls="false"
              class="number-input"
            />
          </template>
          <template v-else>
            <el-input
              v-model="formModel.default_value"
              :disabled="isReadOnly"
              placeholder="请输入默认值"
            />
          </template>
        </el-form-item>
        <el-form-item label="状态">
          <el-select v-model="formModel.status" :disabled="isReadOnly">
            <el-option
              v-for="item in statusOptions"
              :key="item.value"
              :label="item.label"
              :value="item.value"
            />
          </el-select>
        </el-form-item>
        <el-form-item label="说明">
          <el-input
            v-model="formModel.description"
            type="textarea"
            :disabled="isReadOnly"
            placeholder="请输入说明"
            rows="3"
          />
        </el-form-item>

        <template v-if="formModel.value_type === 'enum'">
          <div class="enum-editor-title">枚举值表</div>
          <el-table :data="formModel.enums" border size="small" class="enum-editor">
            <el-table-column label="枚举值" min-width="120">
              <template #default="scope">
                <el-input
                  v-model="scope.row.value"
                  :disabled="isReadOnly"
                  placeholder="输入值"
                />
              </template>
            </el-table-column>
            <el-table-column label="说明" min-width="150">
              <template #default="scope">
                <el-input
                  v-model="scope.row.label"
                  :disabled="isReadOnly"
                  placeholder="输入说明"
                />
              </template>
            </el-table-column>
            <el-table-column label="状态" width="100" align="center">
              <template #default="scope">
                <el-switch
                  v-model="scope.row.status"
                  :disabled="isReadOnly"
                  active-value="enabled"
                  inactive-value="disabled"
                  active-text="在用"
                  inactive-text="停用"
                />
              </template>
            </el-table-column>
            <el-table-column v-if="!isReadOnly" label="操作" width="80" align="center">
              <template #default="scope">
                <el-button
                  link
                  type="danger"
                  size="small"
                  @click="removeEnum(scope.$index)"
                >删除</el-button>
              </template>
            </el-table-column>
          </el-table>
          <div v-if="!isReadOnly" class="enum-ops">
            <el-button type="primary" link @click="addEnum">+ 新增枚举值</el-button>
          </div>
        </template>
      </el-form>

      <template #footer>
        <div class="drawer-footer">
          <el-button @click="closeDrawer">关闭</el-button>
          <template v-if="!isReadOnly">
            <el-button @click="handleReset">重置</el-button>
            <el-button type="primary" @click="handleSave">保存</el-button>
          </template>
        </div>
      </template>
    </el-drawer>
  </div>
</template>

<script setup>
import { computed, reactive, ref, watch, nextTick } from 'vue'
import { ElMessage, ElMessageBox } from 'element-plus'

// 选项常量
const levelOptions = [
  { label: '区域（zone）', value: 'zone' },
  { label: '场所（location）', value: 'location' }
]
const inheritStrategyOptions = [
  { label: '继承父级 (inherit)', value: 'inherit' },
  { label: '必须等于父级 (equal)', value: 'equal' },
  { label: '完全独立 (independent)', value: 'independent' }
]
const valueTypeOptions = [
  { label: '枚举 (enum)', value: 'enum' },
  { label: '文本 (text)', value: 'text' },
  { label: '数字 (number)', value: 'number' },
  { label: '布尔 (bool)', value: 'bool' }
]
const statusOptions = [
  { label: '启用', value: 'enabled' },
  { label: '停用', value: 'disabled' }
]

// Tag 显示映射
const inheritTagTypeMap = {
  inherit: 'success',
  equal: 'warning',
  independent: 'info'
}
const inheritStrategyLabelMap = {
  inherit: '继承父级',
  equal: '必须等于父级',
  independent: '独立配置'
}
const valueTypeTagMap = {
  enum: 'primary',
  text: 'info',
  number: 'warning',
  bool: 'success'
}
const valueTypeLabelMap = {
  enum: '枚举',
  text: '文本',
  number: '数字',
  bool: '布尔'
}
const levelLabelMap = {
  zone: '区域',
  location: '场所'
}

// 模拟属性字典数据
const attributeList = ref([
  {
    id: 1,
    name: '温度阈值',
    key: 'temperature_threshold',
    levels: ['zone'],
    inherit_strategy: 'inherit',
    value_type: 'number',
    status: 'enabled',
    default_value: 26,
    description: '用于空调策略的默认温度阈值',
    enums: [],
    usage_count: 18
  },
  {
    id: 2,
    name: '启用监控',
    key: 'enable_monitor',
    levels: ['zone', 'location'],
    inherit_strategy: 'equal',
    value_type: 'bool',
    status: 'enabled',
    default_value: true,
    description: '是否启用实时监控能力',
    enums: [],
    usage_count: 42
  },
  {
    id: 3,
    name: '巡检频率',
    key: 'inspection_frequency',
    levels: ['location'],
    inherit_strategy: 'independent',
    value_type: 'enum',
    status: 'enabled',
    default_value: 'weekly',
    description: '人工巡检周期设置',
    enums: [
      { value: 'daily', label: '每日', status: 'enabled' },
      { value: 'weekly', label: '每周', status: 'enabled' },
      { value: 'monthly', label: '每月', status: 'disabled' }
    ],
    usage_count: 9
  },
  {
    id: 4,
    name: '楼层备注',
    key: 'floor_note',
    levels: ['location'],
    inherit_strategy: 'inherit',
    value_type: 'text',
    status: 'disabled',
    default_value: '',
    description: '补充楼层特定说明，停用状态仅保留历史',
    enums: [],
    usage_count: 3
  },
  {
    id: 5,
    name: '温湿度模式',
    key: 'comfort_profile',
    levels: ['zone', 'location'],
    inherit_strategy: 'independent',
    value_type: 'enum',
    status: 'enabled',
    default_value: 'standard',
    description: '温湿度控制模式预设',
    enums: [
      { value: 'standard', label: '标准', status: 'enabled' },
      { value: 'energy_saving', label: '节能', status: 'enabled' }
    ],
    usage_count: 27
  }
])

// 筛选状态
const filters = reactive({
  search: '',
  levels: [],
  strategies: [],
  valueTypes: [],
  statuses: []
})

const pagination = reactive({
  currentPage: 1,
  pageSize: 10
})

const emptyText = '暂无属性，点击右上角【+ 新建属性】'

const filteredAttributes = computed(() => {
  return attributeList.value.filter((item) => {
    const searchMatch =
      !filters.search ||
      item.name.toLowerCase().includes(filters.search.toLowerCase()) ||
      item.key.toLowerCase().includes(filters.search.toLowerCase())
    const levelMatch =
      filters.levels.length === 0 ||
      filters.levels.every((level) => item.levels.includes(level))
    const strategyMatch =
      filters.strategies.length === 0 || filters.strategies.includes(item.inherit_strategy)
    const typeMatch = filters.valueTypes.length === 0 || filters.valueTypes.includes(item.value_type)
    const statusMatch = filters.statuses.length === 0 || filters.statuses.includes(item.status)
    return searchMatch && levelMatch && strategyMatch && typeMatch && statusMatch
  })
})

const pagedAttributes = computed(() => {
  const start = (pagination.currentPage - 1) * pagination.pageSize
  return filteredAttributes.value.slice(start, start + pagination.pageSize)
})

watch(
  filters,
  () => {
    pagination.currentPage = 1
  },
  { deep: true }
)

// 抽屉与表单
const drawerVisible = ref(false)
const drawerMode = ref('view') // view | edit | create
const formModel = reactive(createEmptyForm())
const originalSnapshot = ref(null)
const drawerFormRef = ref(null)
const keyError = ref('')
let skipTypeWatch = false

function createEmptyForm() {
  return {
    id: null,
    name: '',
    key: '',
    value_type: 'text',
    levels: [],
    inherit_strategy: 'inherit',
    default_value: '',
    status: 'enabled',
    description: '',
    enums: []
  }
}

const isReadOnly = computed(() => drawerMode.value === 'view')

const drawerTitle = computed(() => {
  if (drawerMode.value === 'create') {
    return '新建属性'
  }
  if (drawerMode.value === 'edit') {
    return `${formModel.name || ''}（编辑）`
  }
  return `${formModel.name || ''}（详情）`
})

const usableEnumOptions = computed(() => {
  if (formModel.value_type !== 'enum') return []
  return formModel.enums.filter((item) => item.status === 'enabled' && item.value && item.label)
})

watch(
  () => formModel.value_type,
  (newType, oldType) => {
    if (skipTypeWatch) return
    if (newType === 'enum') {
      if (!formModel.enums.length) {
        formModel.enums.push({ value: '', label: '', status: 'enabled' })
      }
      if (oldType !== 'enum') {
        formModel.default_value = ''
      }
    } else {
      if (oldType === 'enum') {
        formModel.enums.splice(0, formModel.enums.length)
      }
      if (newType === 'bool') {
        formModel.default_value = true
      } else if (newType === 'number') {
        formModel.default_value = 0
      } else {
        formModel.default_value = ''
      }
    }
  }
)

function handleFilterChange() {
  // 输入事件已通过 watch 处理分页重置，这里仅占位
}

function openView(row) {
  drawerMode.value = 'view'
  openDrawerWithData(row)
}

function openEdit(row) {
  drawerMode.value = 'edit'
  openDrawerWithData(row)
}

function openCreate() {
  drawerMode.value = 'create'
  skipTypeWatch = true
  Object.assign(formModel, createEmptyForm())
  formModel.enums = []
  originalSnapshot.value = JSON.parse(JSON.stringify(formModel))
  keyError.value = ''
  drawerVisible.value = true
  nextTick(() => {
    skipTypeWatch = false
  })
}

function openDrawerWithData(row) {
  skipTypeWatch = true
  Object.assign(formModel, JSON.parse(JSON.stringify(row)))
  if (!Array.isArray(formModel.enums)) {
    formModel.enums = []
  }
  originalSnapshot.value = JSON.parse(JSON.stringify(formModel))
  keyError.value = ''
  drawerVisible.value = true
  nextTick(() => {
    skipTypeWatch = false
  })
}

function closeDrawer() {
  drawerVisible.value = false
}

function handleReset() {
  if (!originalSnapshot.value) return
  skipTypeWatch = true
  Object.assign(formModel, JSON.parse(JSON.stringify(originalSnapshot.value)))
  keyError.value = ''
  nextTick(() => {
    skipTypeWatch = false
  })
}

function addEnum() {
  formModel.enums.push({ value: '', label: '', status: 'enabled' })
}

function removeEnum(index) {
  if (formModel.enums.length <= 1) {
    ElMessage.warning('至少保留一条枚举值，可修改内容或停用')
    return
  }
  formModel.enums.splice(index, 1)
}

function handleSave() {
  if (!validateForm()) return
  if (drawerMode.value === 'create') {
    const newId = Date.now()
    attributeList.value.unshift({
      ...JSON.parse(JSON.stringify(formModel)),
      id: newId,
      usage_count: 0
    })
    ElMessage.success('保存成功，已新增属性')
  } else if (drawerMode.value === 'edit') {
    const index = attributeList.value.findIndex((item) => item.id === formModel.id)
    if (index !== -1) {
      attributeList.value[index] = {
        ...attributeList.value[index],
        ...JSON.parse(JSON.stringify(formModel))
      }
      ElMessage.success('保存成功，已更新属性')
    }
  }
  originalSnapshot.value = JSON.parse(JSON.stringify(formModel))
  drawerVisible.value = false
}

function validateForm() {
  keyError.value = ''
  if (!formModel.name.trim()) {
    ElMessage.error('请填写属性名称')
    return false
  }
  if (!formModel.key.trim()) {
    keyError.value = '请填写属性键名'
    ElMessage.error('请填写属性键名')
    return false
  }
  const keyReg = /^[a-z][a-z0-9_]*$/
  if (!keyReg.test(formModel.key)) {
    keyError.value = '键名需以小写字母开头，可包含数字与下划线'
    ElMessage.error('属性键名格式不正确')
    return false
  }
  const duplicateName = attributeList.value.some(
    (item) => item.name === formModel.name && item.id !== formModel.id
  )
  if (duplicateName) {
    ElMessage.error('属性名称已存在，请使用其他名称')
    return false
  }
  const duplicateKey = attributeList.value.some(
    (item) => item.key === formModel.key && item.id !== formModel.id
  )
  if (duplicateKey) {
    keyError.value = '属性键名已存在'
    ElMessage.error('属性键名已存在')
    return false
  }
  if (!formModel.levels.length) {
    ElMessage.error('请至少选择一个适用层级')
    return false
  }
  if (formModel.value_type === 'enum') {
    if (!formModel.enums.length) {
      ElMessage.error('请至少维护一条枚举值')
      return false
    }
    const validEnums = formModel.enums.filter(
      (item) => item.value && item.label && item.status === 'enabled'
    )
    if (!validEnums.length) {
      ElMessage.error('至少保留一条启用状态的枚举值')
      return false
    }
  }
  return true
}

function handleToggleStatus(row) {
  const action = row.status === 'enabled' ? '停用' : '启用'
  ElMessageBox.confirm(`确认要${action}「${row.name}」吗？`, '提示', {
    type: 'warning',
    confirmButtonText: '确定',
    cancelButtonText: '取消'
  })
    .then(() => {
      row.status = row.status === 'enabled' ? 'disabled' : 'enabled'
      ElMessage.success(`${action}成功`)
    })
    .catch(() => {})
}
</script>

<style scoped>
.attribute-page {
  padding: 16px;
  background: #f5f6f7;
  min-height: 100vh;
  box-sizing: border-box;
}

.toolbar-card {
  margin-bottom: 16px;
}

.toolbar {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 12px;
}

.toolbar-form {
  flex: 1;
}

.table-card {
  margin-bottom: 16px;
}

.attribute-table :deep(.el-table__cell) {
  vertical-align: middle;
}

.mr-4 {
  margin-right: 4px;
  margin-bottom: 4px;
}

.monospace {
  font-family: 'SFMono-Regular', Consolas, 'Liberation Mono', Menlo, monospace;
}

.pagination-wrap {
  margin-top: 16px;
  display: flex;
  justify-content: flex-end;
}

.enum-table-title {
  font-weight: 600;
  margin-bottom: 8px;
}

.enum-empty {
  color: #909399;
  font-size: 13px;
}

.drawer-form {
  padding-right: 8px;
}

.enum-editor-title {
  margin: 16px 0 8px;
  font-weight: 600;
}

.enum-editor {
  margin-bottom: 8px;
}

.enum-ops {
  text-align: right;
}

.drawer-footer {
  text-align: right;
}

.number-input {
  width: 100%;
}
</style>
