<template>
    <div class="invites-admin">
      <el-table :data="sortedInvites" style="width: 100%" border>
        <el-table-column prop="userPhone" label="邀请人手机号" width="180"></el-table-column>
        <el-table-column prop="inviteCode" label="邀请码" width="180"></el-table-column>
        <el-table-column prop="inviteCount" label="成功邀请人数" width="180"></el-table-column>
      </el-table>
    </div>
  </template>
  
  <script>
  import db from '@/api/database'; // 导入数据库实例
  
  export default {
    name: 'InvitesAdmin',
    data() {
      return {
        invites: [],
      };
    },
    computed: {
      sortedInvites() {
        // 按邀请人数降序排列
        return this.invites.sort((a, b) => b.inviteCount - a.inviteCount);
      },
    },
    mounted() {
      this.fetchInvites();
    },
    methods: {
      fetchInvites() {
        db.collection('invite').get().then(res => {
          this.invites = res.data.map(invite => ({
            userPhone: invite.data.userPhone,
            inviteCode: invite.data.inviteCode,
            inviteCount: invite.data.invites.length, // 邀请人数
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
  