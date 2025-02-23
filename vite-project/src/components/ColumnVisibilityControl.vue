<template>
  <div class="column-control">
    <el-button @click="showControl = true;" type="primary">
      自定义表头
    </el-button>

    <el-dialog v-model="showControl" title="列配置" width="600px">
      <div v-for="col in currentColumns" :key="col.prop">
        <div class="column-item">
          <el-checkbox v-model="col.visible">{{ col.label }}</el-checkbox>
          <el-input-number v-if="col.visible" v-model="col.width" :min="50" :max="500" style="margin-left: 16px"
            placeholder="列宽" />
          <el-checkbox v-if="col.visible" v-model="col.fixed" style="margin-left: 16px">
            固定
          </el-checkbox>
        </div>
      </div>

      <template #footer>
        <el-button @click="resetToDefault">恢复默认</el-button>
        <el-button type="primary" @click="savePreferences">保存</el-button>
      </template>
    </el-dialog>
  </div>
</template>

<script setup lang="ts">
import { ref, watch, onMounted } from "vue";
import { ElMessage } from "element-plus";

const props = defineProps({
  columns: {
    type: Array,
    default: () => [],
    validator: (value) => value.every((col) => "prop" in col && "label" in col),
  },
  storageKey: {
    type: String,
    default: "table-columns-preference",
  },
});

const emit = defineEmits(["columns-change"]);

const showControl = ref(false);
const currentColumns = ref([]);
const defaultColumns = ref([]);

// 初始化列数据
const initializeColumns = () => {

  // 每次初始化时都更新 defaultColumns，确保 "恢复默认" 有最新的表头
  defaultColumns.value = JSON.parse(JSON.stringify(props.columns));
  currentColumns.value = JSON.parse(JSON.stringify(props.columns));

  const savedData = localStorage.getItem(props.storageKey);
  if (savedData) {
    try {
      const parsed = JSON.parse(savedData);
      applySavedPreferences(parsed);
    } catch {
      localStorage.removeItem(props.storageKey);
    }
  }

  // 触发 `columns-change`，同步到主组件
  updateVisibleColumns();
};

// 应用用户保存的列配置
const applySavedPreferences = (savedData) => {
  currentColumns.value = props.columns
    .map((col) => {
      const savedCol = savedData.find((c) => c.prop === col.prop);
      return savedCol ? { ...col, ...savedCol } : col;
    })
    .sort((a, b) => a.order - b.order);
};

// 更新可见列
const updateVisibleColumns = () => {
  const visibleColumns = currentColumns.value
    .filter((col) => col.visible)
    .map((col) => ({
      prop: col.prop,
      label: col.label,
      width: col.width,
      fixed: col.fixed,
      visible: col.visible,
    }));
  emit("columns-change", visibleColumns);
};

// 保存用户设置
const savePreferences = () => {
  const dataToSave = currentColumns.value.map((col) => ({
    prop: col.prop,
    visible: col.visible,
    order: col.order,
    width: col.width,
    fixed: col.fixed,
  }));
  localStorage.setItem(props.storageKey, JSON.stringify(dataToSave));
  showControl.value = false;
  ElMessage.success("配置已保存");
};

// 恢复默认
const resetToDefault = () => {
  currentColumns.value = JSON.parse(JSON.stringify(defaultColumns.value));
  localStorage.removeItem(props.storageKey);
  ElMessage.success("已恢复默认配置");
  updateVisibleColumns(); // 确保同步到主组件
};

// 监听 `columns` 的变化**，确保 `currentColumns` 及时更新
watch(
  () => props.columns,
  (newColumns) => {
    initializeColumns(); // 重新同步数据
  },
  { deep: true, immediate: true } // `immediate: true` 确保组件挂载时立即执行
);

// 监听 `currentColumns` 的变化，自动更新可见列
watch(
  currentColumns,
  () => {
    updateVisibleColumns();
  },
  { deep: true }
);

// 组件挂载时初始化
onMounted(() => {
  initializeColumns();
});
</script>

<style scoped>
.column-item {
  padding: 8px 12px;
  margin: 4px 0;
  background: #f5f7fa;
  border-radius: 4px;
  cursor: move;
  display: flex;
  align-items: center;
}
</style>
