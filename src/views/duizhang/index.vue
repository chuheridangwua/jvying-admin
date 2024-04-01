<template>
  <div style="margin-left: 20px;">
    <el-upload style="margin-bottom: 20px;margin-top: 20px;" action="" :auto-upload="false"
      :before-upload="handleBeforeUpload" :on-change="handleChange" ref="upload">
      <el-button slot="trigger" type="primary">选择结算单</el-button>
      <el-button style="margin-left: 20px;" type="primary" @click="uploadToDatabase"
        v-if="isFileSelected">确定上传</el-button>
      <el-date-picker style="margin-left: 20px;" v-model="selectedDate" type="date" format="yyyy 年 MM 月 dd 日"
        value-format="yyyy-MM-dd" placeholder="打款日期" @change="handleDateChange">
      </el-date-picker>
    </el-upload>
    <el-table :data="tableData" style="width: 100%" border>
      <el-table-column prop="sequenceNumber" label="序号" width="70"></el-table-column>
      <el-table-column prop="createTime" label="创建时间" width="160"></el-table-column>
      <el-table-column prop="orderNumber" label="订单号" width="180"></el-table-column>
      <el-table-column prop="transactionId" label="流水号" width="180"></el-table-column>
      <el-table-column prop="accountNumber" label="收款账号" width="120"></el-table-column>
      <el-table-column prop="name" label="姓名" width="90"></el-table-column>
      <el-table-column prop="amountYuan" label="金额（元）" width="120"></el-table-column>
      <el-table-column prop="status" label="状态" width="100">
        <template slot-scope="scope">
          <el-tag :type="scope.row.status === '成功' ? 'success' : 'danger'">
            {{ scope.row.status }}
          </el-tag>
        </template>
      </el-table-column>
      <el-table-column prop="remark" label="备注"></el-table-column>
      <el-table-column prop="failureReason" label="失败原因"></el-table-column>
    </el-table>
  </div>
</template>

<script>
import * as XLSX from 'xlsx';
import db from '@/api/database';

export default {
  data() {
    return {
      tableData: [],
      selectedDate: '', // 存储选定的日期
      isFileSelected: false, // 文件是否被选择的标志
    };
  },
  methods: {
    handleBeforeUpload(file) {
      this.isFileSelected = true; // 更新标志
      const reader = new FileReader();
      reader.onload = (e) => {
        const data = new Uint8Array(e.target.result);
        const workbook = XLSX.read(data, { type: 'array' });
        const sheetName = workbook.SheetNames[0];
        const worksheet = workbook.Sheets[sheetName];
        const jsonData = XLSX.utils.sheet_to_json(worksheet, { header: 1 });
        this.parseData(jsonData);
      };

      // 直接读取 file.raw，如果它是一个 File 对象
      if (file.raw && file.raw instanceof File) {
        reader.readAsArrayBuffer(file.raw);
      } else {
        // 否则，尝试直接使用 file 作为 Blob
        reader.readAsArrayBuffer(file);
      }

      // 阻止默认上传行为
      return false;
    },


    handleChange(file, fileList) {
      console.log(fileList); // 输出文件列表
      // 确保 fileList 中的文件和你要处理的 file 是一致的
      this.$refs.upload.submit();
    },
    parseData(data) {
      // 从第12行开始解析数据
      const rows = data.slice(11);
      this.tableData = rows.map(row => {
        return {
          sequenceNumber: row[0],   // 序号
          createTime: row[1],       // 创建时间
          orderNumber: row[2],      // 订单号
          transactionId: row[3],    // 流水号
          accountNumber: row[4],    // 收款账号
          name: row[5],             // 姓名
          amountYuan: row[6],       // 金额（元）
          status: row[7],           // 状态
          remark: row[8],           // 备注
          failureReason: row[9]     // 失败原因
        };
      });
    },
    uploadToDatabase() {
      this.$confirm('确定要上传这些数据吗？', '确认上传', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        this.tableData.forEach((item) => {
          db.collection("pay").add({
            data: item
          }).then(res => {
            console.log('Upload successful', res);
            this.$message.success('数据上传成功');
            // 成功上传后更新income集合的状态
            this.updateIncomeStatus(item.accountNumber);
          }).catch(err => {
            console.error('Upload failed', err);
            this.$message.error('数据上传失败');
          });
        });
      }).catch(() => {
        this.$message.info('取消上传');
      });
    },
    updateIncomeStatus(accountNumber) {
      // 确保accountNumber是一个字符串
      const accountNumberStr = accountNumber.toString();

      db.collection("income").where({
        "data.payPhone": accountNumberStr, // 使用字符串进行匹配
      }).get().then(res => {
        if (res.data.length > 0) {
          // 创建一个promise数组来存储所有更新操作
          const updatePromises = res.data.map(doc => {
            return db.collection("income").doc(doc._id).update({
              data: {
                status: '已打款'
              }
            });
          });

          // 使用Promise.all等待所有更新操作完成
          Promise.all(updatePromises).then(() => {
            console.log(`All statuses updated for accountNumber: ${accountNumberStr}`);
            this.$message.success(`所有状态已更新为已打款。`);
          }).catch(err => {
            console.error(`Error updating statuses for accountNumber: ${accountNumberStr}`, err);
            this.$message.error('更新状态时发生错误。');
          });
        } else {
          console.log(`No matching documents found for accountNumber: ${accountNumberStr}`);
          this.$message.info(`未找到匹配的文档: ${accountNumberStr}`);
        }
      }).catch(err => {
        console.error('Query failed', err);
        this.$message.error('查询失败。');
      });
    },


    handleDateChange(value) {
      // 处理日期选择后的逻辑
      // 这里以字符串形式展示选定的日期
      console.log('Selected date: ', value);
      // 根据选定的日期查询数据库
      this.fetchDataForDate(value);
    },
    fetchDataForDate(selectedDate) {
      console.log('fetchDataForDate', selectedDate);
      if (!selectedDate) {
        this.$message.warning('请选择一个日期');
        return;
      }

      let startDate = selectedDate + " 00:00:00";
      let endDate = selectedDate + " 23:59:59";

      db.collection("pay").where({
        "data.createTime": db.command.gte(startDate).and(db.command.lte(endDate))
      }).get().then(res => {
        if (res.data.length === 0) {
          this.tableData = []; // 如果没有数据，设置tableData为一个空数组
          this.$message.info('所选日期没有数据');
        } else {
          console.log("res", res)
          // 假设查询结果在res.data中，并且每个结果的实际数据位于其data属性
          const formattedData = res.data.map(item => ({
            sequenceNumber: item.data.sequenceNumber,
            createTime: item.data.createTime,
            orderNumber: item.data.orderNumber,
            transactionId: item.data.transactionId,
            accountNumber: item.data.accountNumber,
            name: item.data.name,
            amountYuan: item.data.amountYuan,
            status: item.data.status,
            remark: item.data.remark,
            failureReason: item.data.failureReason // 假设这是可选的，只在有失败原因时存在
          }));

          // 更新tableData以反映查询结果
          this.tableData = formattedData;
        }

      }).catch(err => {
        console.error('Query failed', err);
        this.$message.error('查询失败');
      });
    },


  }
};
</script>
