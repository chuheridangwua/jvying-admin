<template>
  <div class="invites-admin">
    <el-table :data="integrals" style="width: 100%" border>
      <el-table-column prop="userName" label="用户姓名" width="120"></el-table-column>
      <el-table-column prop="userPhone" label="用户手机号" width="120"></el-table-column>
      <el-table-column prop="productName" label="兑换商品名称" width="180"></el-table-column>
      <el-table-column prop="consumedPoints" label="消耗积分" width="100"></el-table-column>
      <el-table-column prop="exchangeTime" label="兑换时间" width="180"></el-table-column>
    </el-table>
  </div>
</template>

<script>
import db from '@/api/database'; // 导入数据库实例

export default {
  name: 'InvitesAdmin',
  data() {
    return {
      integrals: [],
    };
  },
  mounted() {
    this.fetchInvites();
  },
  methods: {
    fetchInvites() {
      db.collection('integral').get().then(res => {
        console.log(res)
        this.integrals = res.data.map(item => ({
          userName: item.data.userName,
          userPhone: item.data.userPhone,
          productName: item.data.productName,
          consumedPoints: item.data.consumedPoints,
          exchangeTime: item.data.exchangeTime
        }));
      }).catch(err => {
        console.error('Error fetching invites:', err);
      });
    },
  },
};
</script>

<style scoped>
.invites-admin {
  padding: 20px;
}
</style>
