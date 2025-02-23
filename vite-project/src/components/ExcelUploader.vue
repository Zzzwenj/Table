<template>
    <el-upload action="" :http-request="httpExcelRequest" :limit="1" :show-file-list="false" class="uploadExcelContent">
        <el-button type="primary">导入Excel</el-button>
    </el-upload>
</template>

<script setup lang="ts">
import { ref, defineEmits } from "vue";
import * as xlsx from "xlsx";
import { ElMessage } from "element-plus";

// 定义组件事件
const emit = defineEmits(["update:columns", "update:data"]);

// 定义正则表达式规则（假设列的规则）
const columnValidationRules = {
    "序号": /^[0-9]+$/, // 序号必须是数字
    "名称": /.+/, // 名称不能为空
    "金额": /^\d+(\.\d{1,2})?$/, // 金额格式为数字，最多两位小数
    "状态": /^(进行中|已完成|待审批)$/, // 状态只能是这三种，并且不能为空
};

// 解析 Excel 文件
const httpExcelRequest = async (op) => {
    let file = op.file;
    try {
        let dataBinary = await readFile(file);
        let workBook = xlsx.read(dataBinary, { type: "binary", cellDates: true });
        let workSheet = workBook.Sheets[workBook.SheetNames[0]];
        const excelData = xlsx.utils.sheet_to_json(workSheet, { header: 1 });

        if (excelData.length === 0) {
            ElMessage.error("Excel 文件为空");
            return;
        }

        // 解析表头
        const headers = excelData[0];

        // 检查是否包含"名称"列
        if (!headers.includes("名称")) {
            ElMessage.error("Excel 文件中必须包含 '名称' 列");
            return;
        }
        
        const tableColumns = headers.map(header => ({
            prop: header,
            label: header,
            visible: true,
            width: 120,
            fixed: false
        }));

        // 解析数据并进行验证
        const tableData = excelData.slice(1).map(row => {
            let rowData = {};
            let validationError = null;

            headers.forEach((header, index) => {
                const value = row[index] || "";
                rowData[header] = value;

                // 校验每列数据
                if (columnValidationRules[header]) {
                    const rule = columnValidationRules[header];
                    if (!rule.test(value)) {
                        validationError = `${header} 数据是必填项或者数据格式不正确`;
                    }
                }
            });

            if (validationError) {
                ElMessage.error(validationError);
                return null; // 该行数据不符合规则，返回 null
            }

            return rowData; // 返回符合规则的数据行
        }).filter(row => row !== null); // 删除无效的数据行

        if (tableData.length > 0) {
            // 触发事件传递表头和数据
            emit("update:columns", tableColumns);
            emit("update:data", tableData);
        } else {
            ElMessage.error("没有有效数据被导入");
        }
    } catch (error) {
        console.error("文件读取或解析失败:", error);
        ElMessage.error("文件读取或解析失败");
    }
};

// 读取文件内容
const readFile = (file) => {
    return new Promise((resolve, reject) => {
        let reader = new FileReader();
        reader.readAsBinaryString(file);
        reader.onload = (ev) => {
            resolve(ev.target?.result);
        };
        reader.onerror = (error) => {
            reject(error);
        };
    });
};
</script>

<style scoped>
.uploadExcelContent {
    margin-bottom: 20px;
}
</style>