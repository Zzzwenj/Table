<template>
    <div class="my-table">
        <h2>表格组件</h2>
        <!-- 表格 -->
        <el-table ref="tableRef" :data="loadedData" border :key="tableKey" row-key="tableKey" :span-method="mergeCells"
            height="400px" header-row-class-name="fixed-header" @scroll="handleScroll">
            <!-- 拖拽手柄列 -->
            <el-table-column width="60" fixed="left" v-if="!isMerged">
                <template #default >
                    <span class="drag-handle" style="cursor: move;">|||</span>
                </template>
            </el-table-column>
            <el-table-column v-for="col in visibleColumns" :key="col.prop" :prop="col.prop" :label="col.label"
                :width="col.width" :fixed="col.fixed" />
        </el-table>

        <div class="botton-group">
            <!-- Excel 上传 -->
            <ExcelUploader @update:columns="updateColumns" @update:data="updateData" />

            <!-- 列配置 -->
            <column-control :columns="tableColumns" storage-key="user-table-preferences"
                @columns-change="handleColumnsChange" />

            <!-- 是否合并单元格 -->
            <el-button type="primary" @click="toggleMerge" class="merge-button">
                {{ isMerged ? "取消合并" : "合并单元格" }}
            </el-button>

        </div>
    </div>
</template>

<script setup lang="ts">
import { ref, watch, nextTick, onMounted } from "vue";
import ColumnControl from "./ColumnVisibilityControl.vue";
import ExcelUploader from "./ExcelUploader.vue";
import Sortable from 'sortablejs';

// 是否合并单元格
const isMerged = ref(false);
const tableRef = ref();
let sortableInstance: Sortable | null = null;

// 数据管理
const allData = ref<Array<Record<string, any>>>([]);
const loadedData = ref<Array<Record<string, any>>>([]);
const currentPage = ref(0);
const pageSize = ref(50);
const loadingThreshold = ref(20);
const isLoading = ref(false);

// 列配置
const tableColumns = ref([]);
const visibleColumns = ref([]);
const tableKey = ref(0);

// 添加滚动位置记录
const lastScrollTop = ref(0);

// 初始化拖拽功能
let initPending = false;
const initSortable = () => {
    if (initPending) return;
    initPending = true;
    const table = tableRef.value;
    if (!table) return;

    // 销毁旧的实例
    if (sortableInstance) {
        sortableInstance.destroy();
        sortableInstance = null;
    }

    const tbody = table.$el.querySelector('.el-table__body-wrapper tbody');
    if (tbody) {
        sortableInstance = Sortable.create(tbody, {
            animation: 150,
            handle: '.drag-handle',
            onEnd: ({ oldIndex, newIndex }) => {
                console.log('拖拽结束', oldIndex, newIndex);

                if (oldIndex === undefined || newIndex === undefined) return;
                const newData = [...loadedData.value];
                const [movedRow] = newData.splice(oldIndex, 1);
                newData.splice(newIndex, 0, movedRow);
                loadedData.value = newData;
                tableKey.value += 1;
            }
        });
    }
    nextTick(() => {
        initPending = false;
    });
};

// 监听数据变化重新初始化拖拽
watch(loadedData, () => {
    nextTick(() => {
        initSortable();
    });
});

// 组件挂载时初始化
onMounted(() => {
    initSortable();
});

// 获取滚动容器
const getScrollContainer = () => {
    if (!tableRef.value) return null;
    return tableRef.value.$el.querySelector('.el-scrollbar__wrap');
};

// 滚动处理
let scrollTimeout: any = null;
const handleScroll = () => {
    if (scrollTimeout) {
        clearTimeout(scrollTimeout);
    }

    scrollTimeout = setTimeout(() => {
        const scrollContainer = getScrollContainer();
        if (!scrollContainer || isLoading.value) return;

        const { scrollTop, scrollHeight, clientHeight } = scrollContainer;
        lastScrollTop.value = scrollTop; // 更新最后的滚动位置

        if (scrollHeight - scrollTop - clientHeight < clientHeight * 0.2) {
            loadMore();
        }
    }, 100); // 100ms 的防抖延迟
};

// 加载更多数据
const loadMore = async () => {
    if (isLoading.value) return;
    isLoading.value = true;

    try {
        // 保存当前滚动位置
        const scrollContainer = getScrollContainer();
        const currentScrollTop = scrollContainer?.scrollTop || 0;

        const start = currentPage.value * pageSize.value;
        const end = start + pageSize.value;
        const newData = allData.value.slice(start, end);

        if (newData.length === 0) {
            console.log('已加载全部数据');
            isLoading.value = false;
            return;
        }

        console.log(`加载数据: ${start + 1} - ${end} 行`);

        // 直接更新数据，不触发 tableKey 的变化
        loadedData.value = [...loadedData.value, ...newData];
        currentPage.value++;

        // 使用 nextTick 确保在数据更新后恢复滚动位置
        await nextTick();
        const newScrollContainer = getScrollContainer();
        if (newScrollContainer) {
            newScrollContainer.scrollTop = currentScrollTop;
        }
    } finally {
        isLoading.value = false;
    }
};

// 合并单元格逻辑（基于已加载的数据）
const mergeCells = ({ row, column, rowIndex }) => {
    if (!isMerged.value) return [1, 1];
    const prop = column.property;
    if (!prop) return [1, 1];

    const mergeColumns = tableColumns.value.map(col => col.prop);
    if (!mergeColumns.includes(prop)) return [1, 1];

    let rowspan = 1;
    if (rowIndex > 0 && loadedData.value[rowIndex - 1][prop] === row[prop]) {
        return [0, 0];
    }

    for (let i = rowIndex + 1; i < loadedData.value.length; i++) {
        if (loadedData.value[i][prop] === row[prop]) {
            rowspan++;
        } else {
            break;
        }
    }

    return [rowspan, 1];
};

// 数据更新处理
const updateData = (data: Array<Record<string, any>>) => {
    allData.value = data.map((item, index) => ({
        ...item,
        globalIndex: index,
    }));
    loadedData.value = [];
    currentPage.value = 0;
    loadMore();
};

// 其他原有方法
const toggleMerge = () => {
    isMerged.value = !isMerged.value;
    // 只在切换合并状态时更新 tableKey
    tableKey.value += 1;
};

const indexMethod = (index: number) => index;

const updateColumns = (columns: any[]) => {
    tableColumns.value = columns;
    handleColumnsChange(columns);
};

const handleColumnsChange = (visibleCols) => {
    visibleColumns.value = visibleCols.filter(col => col.visible === true);
    tableKey.value += 1;
};

// 初始化列配置监听
watch(tableColumns, (newColumns) => {
    handleColumnsChange(newColumns);
});
</script>

<style scoped>
.my-table {
    display: flex;
    flex-direction: column;
    gap: 10px;
}

.botton-group {
    display: flex;
    justify-content: flex-end;
    margin-top: 20px;
    gap: 10px;
}

.merge-button {
    width: 100px;
}

/* 拖拽手柄样式 */
.drag-handle {
    color: #999;
    font-size: 12px;
    padding: 0 8px;
}
</style>