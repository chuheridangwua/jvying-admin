<template>
  <div class="settlement-page">
    <el-button @click="generateSettlement" type="success" style="margin-top: 20px;">生成结算单</el-button>
    <el-table :data="filteredData" style="width: 100%; margin-top: 20px;">
      <el-table-column prop="applicantName" label="申请人姓名"></el-table-column>
      <el-table-column prop="payName" label="结算姓名"></el-table-column>
      <el-table-column prop="payPhone" label="结算手机号"></el-table-column>
      <!-- <el-table-column prop="applicationTime" label="申请时间" :formatter="formatDate"></el-table-column> -->
      <el-table-column prop="title" label="项目名称"></el-table-column>
      <el-table-column prop="price" label="金额"></el-table-column>
    </el-table>
  </div>
</template>

<script>
import db from '@/api/database'; // 假设你已经有了数据库的引用
import * as XLSX from 'xlsx';

export default {
  data() {
    return {
      rawData: [],
      filteredData: [],
    };
  },
  created() {
    this.fetchData();
  },
  methods: {
    fetchData() {
      db.collection('income').get().then(res => {
        console.log('加载数据成功:', res.data);
        this.rawData = res.data.map(item => ({ ...item.data, applicationTime: item.data.applicationTime.$date }));
        this.filteredData = this.rawData.filter(item => item.status === '审核通过');
      }).catch(err => {
        console.error('加载数据失败:', err);
      });
    },
    generateSettlement() {
      // 定义标题行，第一行的单元格将被合并
      const titleRow = [['支付宝批量付款文件模板（前面两行请勿删除）']];
      // 定义列头
      const headerRow = [['序号（必填）', '收款方支付宝账号（必填）', '收款方姓名（必填）', '金额（必填，单位：元）', '备注（选填）']];
      // 准备数据行
      const dataRows = this.filteredData.map((item, index) => ([
        index + 1,
        item.payPhone,
        item.payName,
        parseFloat(item.price).toFixed(2), // 确保金额格式正确，保留两位小数
        item.title
      ]));

      // 合并标题行、列头和数据行
      const output = [...titleRow, ...headerRow, ...dataRows];

      // 使用xlsx工具函数生成工作表
      const ws = XLSX.utils.aoa_to_sheet(output);

      // 设置工作表的合并单元格
      ws['!merges'] = [
        // 合并第一行的前五个单元格
        { s: { r: 0, c: 0 }, e: { r: 0, c: headerRow[0].length - 1 } },
      ];

      // 创建新的工作簿，并添加工作表
      const wb = XLSX.utils.book_new();
      XLSX.utils.book_append_sheet(wb, ws, "结算单");

      // 输出工作簿到文件
      XLSX.writeFile(wb, "结算单.xlsx");
    },

    formatDate(row, column, cellValue) {
      // 从对象中获取日期字符串
      const dateString = cellValue && cellValue.$date ? cellValue.$date : cellValue;

      // 使用日期字符串创建一个 Date 对象
      const date = new Date(dateString);

      if (!isNaN(date.getTime())) {
        // 如果是有效日期，格式化并返回
        const year = date.getFullYear();
        const month = (date.getMonth() + 1).toString().padStart(2, '0');
        const day = date.getDate().toString().padStart(2, '0');
        return `${year}-${month}-${day}`;
      } else {
        // 如果日期无效，返回空字符串
        return '';
      }
    },


  }
};
</script>

<style scoped>
.settlement-page {
  margin: 20px;
}
</style>
