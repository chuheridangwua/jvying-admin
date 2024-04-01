<template>
  <div class="withdrawal-backend">
    <el-table :data="incomeList" style="width: 100%">
      <el-table-column prop="applicationTime" label="申请时间" :formatter="formatDate"></el-table-column>
      <el-table-column prop="title" label="项目名称"></el-table-column>
      <el-table-column prop="applicantName" label="申请人姓名"></el-table-column>
      <el-table-column prop="applicantPhone" label="申请人手机号"></el-table-column>
      <el-table-column prop="payName" label="结算姓名"></el-table-column>
      <el-table-column prop="payPhone" label="结算手机号"></el-table-column>
      <el-table-column label="申请周期" :formatter="formatSettlementPeriod"></el-table-column>
      <el-table-column label="审核备注" width="250">
        <template slot-scope="scope">
          <el-input v-model="scope.row.review" placeholder="输入审核备注" @blur="onReviewBlur(scope.row)"></el-input>
        </template>
      </el-table-column>
      <el-table-column label="状态" width="180">
        <template slot-scope="scope">
          <el-select v-model="scope.row.status" placeholder="请选择状态" @change="() => onStatusChange(scope.row)">
            <el-option label="待审核" value="待审核"></el-option>
            <el-option label="审核通过" value="审核通过"></el-option>
            <el-option label="审核驳回" value="审核驳回"></el-option>
            <el-option label="已打款" value="已打款"></el-option>
            <el-option label="已结束" value="已结束"></el-option>
          </el-select>
        </template>
      </el-table-column>
    </el-table>
  </div>
</template>

<script>
import db from '@/api/database';

export default {
  name: 'WithdrawalBackend',
  data() {
    return {
      incomeList: [],
    };
  },
  mounted() {
    this.fetchIncomeList();
  },
  methods: {
    fetchIncomeList() {
      db.collection('income').get().then(res => {
        this.incomeList = res.data.map(item => ({
          ...item.data,
          id: item._id // 保存记录ID以便后续更新
        }));
      }).catch(err => {
        console.error('获取提现申请列表失败:', err);
      });
    },
    onStatusChange(row) {

      this.$confirm(`确定要将状态更新为"${row.status}"吗？`, '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        db.collection('income').doc(row.id).update({
          data: { status: row.status }
        }).then(() => {
          this.$message({
            type: 'success',
            message: '状态更新成功!'
          });
        }).catch(err => {
          ElMessage.error('');
          this.$message({
            type: 'error',
            message: '状态更新失败!'
          });
          console.error('状态更新失败:', err);
        });
      }).catch(() => {
        row.status = this.prevStatus; // 假设 prevStatus 是之前的状态
        this.$message({
          type: 'info',
          message: '已取消'
        });
      });
    },
    formatSettlementPeriod(row) {
      // 假设申请周期保存在 settlementPeriod 字段
      return row.settlementPeriod ? `${row.settlementPeriod[0]} 至 ${row.settlementPeriod[1]}` : '';
    },
    formatDate(row, column, cellValue) {
      const date = new Date(cellValue);
      return date.toLocaleDateString() + ' ' + date.toLocaleTimeString();
    },
    onReviewBlur(row) {
      // 更新审核备注
      db.collection('income').doc(row.id).update({
        data: { review: row.review }
      }).then(() => {
        console.log('审核备注更新成功');
        // 可以在这里添加用户反馈
      }).catch(err => {
        console.error('审核备注更新失败:', err);
      });
    },
  }
};
</script>

<style scoped>
.withdrawal-backend {
  margin: 20px;
}
</style>
