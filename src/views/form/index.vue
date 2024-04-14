<template>
  <div class="user-table">
    <div class="filters" style="margin: 20px;">
      <!-- 第一行筛选器 -->
      <el-row :gutter="10">
        <el-col :span="8">
          <el-input placeholder="搜索姓名" v-model="filters.name" @input="applyFilters" style="width: 100%;"></el-input>
        </el-col>
        <el-col :span="8">
          <el-input placeholder="搜索电话" v-model="filters.phone" @input="applyFilters" style="width: 100%;"></el-input>
        </el-col>
        <el-col :span="8">
          <el-select v-model="filters.community" placeholder="选择小区" @change="applyFilters" style="width: 100%;">
            <el-option v-for="item in uniqueCommunities" :key="item" :label="item" :value="item"></el-option>
          </el-select>
        </el-col>
      </el-row>

      <!-- 第二行筛选器 -->
      <el-row :gutter="10" style="margin-top: 10px;">
        <el-col :span="6">
          <el-select v-model="filters.building" placeholder="选择楼号" @change="applyFilters" style="width: 100%;">
            <el-option v-for="item in uniqueBuildings" :key="item" :label="item" :value="item"></el-option>
          </el-select>
        </el-col>
        <el-col :span="6">
          <el-select v-model="filters.unit" placeholder="选择单元号" @change="applyFilters" style="width: 100%;">
            <el-option v-for="item in uniqueUnits" :key="item" :label="item" :value="item"></el-option>
          </el-select>
        </el-col>
        <el-col :span="6">
          <el-select v-model="filters.room" placeholder="选择门牌号" @change="applyFilters" style="width: 100%;">
            <el-option v-for="item in uniqueRooms" :key="item" :label="item" :value="item"></el-option>
          </el-select>
        </el-col>
        <el-col :span="6">
          <el-select v-model="filters.familyRole" placeholder="选择家庭角色" @change="applyFilters" style="width: 100%;">
            <el-option v-for="role in familyRoles" :key="role" :label="role" :value="role"></el-option>
          </el-select>
        </el-col>
      </el-row>
    </div>


    <el-table :data="tableData" style="width: 100%;margin: 20px;">
      <el-table-column prop="name" label="姓名" sortable></el-table-column>
      <el-table-column prop="phone" label="电话"></el-table-column>
      <el-table-column prop="address.community" label="小区"></el-table-column>
      <el-table-column prop="address.building" label="楼号"></el-table-column>
      <el-table-column prop="address.unit" label="单元号"></el-table-column>
      <el-table-column prop="address.room" label="门牌号"></el-table-column>
      <el-table-column prop="familyRole" label="家庭角色"></el-table-column>
      <el-table-column prop="stars" label="积分" sortable></el-table-column>
      <!-- <el-table-column label="操作">
        <template slot-scope="scope">
          <el-button size="mini" type="primary" @click="viewDetails(scope.row)">查看详情</el-button>
        </template>
      </el-table-column> -->
    </el-table>

    <!-- Details Dialog -->
    <el-dialog :visible.sync="detailsDialogVisible" width="50%">
      <div v-if="userDetails">
        <p>姓名: {{ userDetails.name }}</p>
        <p>电话: {{ userDetails.phone }}</p>
        <p>地址: {{ userDetails.address.community }} {{ userDetails.address.building }}-{{ userDetails.address.unit }}-{{
        userDetails.address.room }}</p>
        <p>家庭角色: {{ userDetails.familyRole }}</p>
        <p>积分: {{ userDetails.stars }}</p>
        <!-- Add more details here as needed -->
      </div>
    </el-dialog>
  </div>
</template>

<script>
import db from '@/api/database';

export default {
  data() {
    return {
      tableData: [],
      filteredData: [],
      filters: {
        name: '',
        phone: '',
        community: '',
        building: '',
        unit: '',
        room: '',
        familyRole: ''
      },
      familyRoles: ['父亲', '母亲', '孩子']
    };
  },
  computed: {
    uniqueCommunities() {
      return [...new Set(this.tableData.map(item => item.address.community))];
    },
    uniqueBuildings() {
      return [...new Set(this.tableData.map(item => item.address.building))];
    },
    uniqueUnits() {
      return [...new Set(this.tableData.map(item => item.address.unit))];
    },
    uniqueRooms() {
      return [...new Set(this.tableData.map(item => item.address.room))];
    }
  },
  mounted() {
    this.fetchData();
  },
  methods: {
    fetchData() {
      db.collection("users-chengjun").get()
        .then(res => {
          this.tableData = res.data;
          this.filteredData = res.data;
        })
        .catch(err => {
          console.error("获取数据失败:", err);
        });
    },
    applyFilters() {
      this.filteredData = this.tableData.filter(item => {
        return (!this.filters.name || item.name.includes(this.filters.name)) &&
          (!this.filters.phone || item.phone.includes(this.filters.phone)) &&
          (!this.filters.community || item.address.community === this.filters.community) &&
          (!this.filters.building || item.address.building === this.filters.building) &&
          (!this.filters.unit || item.address.unit === this.filters.unit) &&
          (!this.filters.room || item.address.room === this.filters.room) &&
          (!this.filters.familyRole || item.familyRole === this.filters.familyRole);
      });
    }
  }
};
</script>
