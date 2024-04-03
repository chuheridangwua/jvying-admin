<template>
  <div>
    <!-- 数据卡片区域 -->
    <div class="data-cards" style="margin: 20px;">
      <div class="data-card" v-for="(card, index) in cards" :key="index">
        <div class="card-content">
          <div class="card-number">{{ card.number }}</div>
          <div class="card-label">{{ card.label }}</div>
        </div>
      </div>
    </div>

    <!-- 折线图区域 -->
    <div>
      <div ref="registrationChart" style="width: 90vw;height:60vh;margin: 20px auto;"></div>
    </div>
  </div>
</template>

<script>
import * as echarts from 'echarts';
import app from '@/api/appwx';
import db from '@/api/database';

export default {
  name: 'RegistrationStatsPage',

  data() {
    const now = new Date();
    const todayDate = now.toISOString().split('T')[0];
    const currentTime = now.toTimeString().split(' ')[0];

    const startOfMonth = new Date(now.getFullYear(), now.getMonth(), 1);
    const startOfMonthDate = startOfMonth.toISOString().split('T')[0];
    const endOfMonthDate = new Date(now.getFullYear(), now.getMonth() + 1, 0).toISOString().split('T')[0];

    return {
      cards: [
        { label: '注册总人数', number: 0, date: '' },
        { label: '今日注册人数', number: 0, date: `今天：${todayDate}` },
        { label: '当日打款金额', number: '0', date: `今天：${todayDate}` },
        { label: '当月打款金额', number: '0', date: `本月：${startOfMonthDate}至${todayDate}` },
        { label: '当日活跃人数', number: 0, date: `今天：${todayDate}` },
        { label: '当月活跃人数', number: 0, date: `本月：${startOfMonthDate}至${endOfMonthDate}` },
      ],
      chartData: {
        dates: [],
        activeUsers: [],
        paymentAmounts: [],
        registrationCounts: [],
      },
    }
  },

  mounted() {
    Promise.all([
      this.fetchUserData(),
      this.fetchRegistrationData(),
      this.fetchDailyPaymentAmount(),
      this.fetchMonthlyPaymentAmount(),
      this.fetchDailyActiveUsers(),
      this.fetchMonthlyActiveUsers(),
    ]).then(() => {
      this.drawRegistrationChart();
    }).catch(err => {
      console.error('Error loading data:', err);
    });

  },

  methods: {
    fetchDailyPaymentAmount() {
      const todayStart = new Date();
      todayStart.setHours(0, 0, 0, 0);
      const todayStartString = todayStart.toISOString().split('T')[0] + " 00:00:00";

      const now = new Date();
      const nowString = now.toISOString().replace('T', ' ').slice(0, 19); // 获取当前时间的字符串，格式为YYYY-MM-DD HH:MM:SS

      db.collection('pay')
        .where({
          "data.createTime": {
            $gte: todayStartString,
            $lte: nowString,
          },
        })
        .get()
        .then(res => {
          const totalAmountToday = res.data.reduce((sum, record) => sum + parseFloat(record.data.amountYuan), 0);
          this.cards[2].number = totalAmountToday.toFixed(2); // 更新当日打款金额卡片
        })
        .catch(err => {
          console.error('Error getting daily payment data:', err);
        });
    },

    fetchMonthlyPaymentAmount() {
      const startOfMonth = new Date(new Date().getFullYear(), new Date().getMonth(), 1);
      const startOfMonthString = startOfMonth.toISOString().split('T')[0] + " 00:00:00";

      const now = new Date();
      const nowString = now.toISOString().replace('T', ' ').slice(0, 19); // 获取当前时间的字符串，格式为YYYY-MM-DD HH:MM:SS

      db.collection('pay')
        .where({
          "data.createTime": {
            $gte: startOfMonthString,
            $lte: nowString,
          },
        })
        .get()
        .then(res => {
          const totalAmountMonth = res.data.reduce((sum, record) => sum + parseFloat(record.data.amountYuan), 0);
          this.cards[3].number = totalAmountMonth.toFixed(2); // 更新当月打款金额卡片
        })
        .catch(err => {
          console.error('Error getting monthly payment data:', err);
        });
    },

    fetchDailyActiveUsers() {
      const todayStart = new Date();
      todayStart.setHours(0, 0, 0, 0);

      db.collection('login')
        .get()
        .then(res => {
          const todayStartStr = `${todayStart.getFullYear()}年${todayStart.getMonth() + 1}月${todayStart.getDate()}日`;
          const dailyActiveUsers = res.data.filter(item => item.data.loginTime.startsWith(todayStartStr));
          this.cards[4].number = dailyActiveUsers.length; // 更新当日活跃人数卡片
        })
        .catch(err => {
          console.error('Error getting daily active user data:', err);
        });
    },

    fetchMonthlyActiveUsers() {
      const startOfMonth = new Date(new Date().getFullYear(), new Date().getMonth(), 1);

      db.collection('login')
        .get()
        .then(res => {
          const monthStartStr = `${startOfMonth.getFullYear()}年${startOfMonth.getMonth() + 1}月`;
          const monthlyActiveUsers = res.data.filter(item => item.data.loginTime.startsWith(monthStartStr));
          this.cards[5].number = monthlyActiveUsers.length; // 更新当月活跃人数卡片
        })
        .catch(err => {
          console.error('Error getting monthly active user data:', err);
        });
    },

    fetchUserData() {
      const todayStart = new Date();
      todayStart.setHours(0, 0, 0, 0);

      const todayEnd = new Date();
      todayEnd.setHours(23, 59, 59, 999);

      db.collection('user')
        .get()
        .then(res => {
          console.log("注册总人数res",res)
          // 更新注册总人数
          this.cards[0].number = res.data.length;

          // 筛选并更新今日注册人数
          const todayRegistrations = res.data.filter(user => {
            // 确保正确解析createTime为日期对象
            const createTime = new Date(user.data.createTime);
            return createTime >= todayStart && createTime <= todayEnd;
          });
          this.cards[1].number = todayRegistrations.length;
        })
        .catch(err => {
          console.error('Error getting user data:', err);
        });
    },

    formatDateCNtoISO(dateStr) {
      const parts = dateStr.match(/(\d+)年(\d+)月(\d+)日/);
      if (!parts) return null; // 如果不匹配，返回null
      // 注意：月份和日期需要补0
      const isoString = `${parts[1]}-${parts[2].padStart(2, '0')}-${parts[3].padStart(2, '0')}`;
      return isoString;
    },

    async fetchRegistrationData() {
      // 初始化日期和计数
      for (let i = 29; i >= 0; i--) {
        const date = new Date();
        date.setDate(date.getDate() - i);
        const dateString = date.toISOString().split('T')[0];
        this.chartData.dates.push(dateString);
        this.chartData.registrationCounts.push(0);
        this.chartData.activeUsers.push(0);
        this.chartData.paymentAmounts.push(0);
      }

      try {
        // 获取注册用户数据
        const userRes = await db.collection('user').get();
        userRes.data.forEach(user => {
          const createTime = new Date(user.data.createTime).toISOString().split('T')[0];
          const index = this.chartData.dates.indexOf(createTime);
          if (index !== -1) {
            this.$set(this.chartData.registrationCounts, index, this.chartData.registrationCounts[index] + 1);
          }
        });

        // 获取活跃用户数据
        const loginRes = await db.collection('login').get();
        loginRes.data.forEach(login => {
          // 对loginTime使用formatDateCNtoISO进行格式化
          const loginTimeISO = this.formatDateCNtoISO(login.data.loginTime.split(' ')[0]); // 只取日期部分，忽略时间
          const index = this.chartData.dates.indexOf(loginTimeISO);
          if (index !== -1) {
            this.$set(this.chartData.activeUsers, index, this.chartData.activeUsers[index] + 1);
          }
        });

        // 获取打款金额数据
        const paymentRes = await db.collection('pay').get();
        paymentRes.data.forEach(payment => {
          const paymentTime = new Date(payment.data.createTime).toISOString().split('T')[0];
          console.log("paymentTime",paymentTime)
          const index = this.chartData.dates.indexOf(paymentTime);
          if (index !== -1) {
            this.$set(this.chartData.paymentAmounts, index, this.chartData.paymentAmounts[index] + parseFloat(payment.data.amountYuan));
          }
        });

        // 数据处理完毕后调用绘图
      } catch (err) {
        console.error('Error getting data:', err);
      } finally {
        this.drawRegistrationChart();
      }
    },

    drawRegistrationChart() {
      const chartDom = this.$refs.registrationChart;
      const myChart = echarts.init(chartDom);
      const option = {
        tooltip: {
          trigger: 'axis'
        },
        legend: {
          data: ['活跃人数', '打款金额', '注册人数']
        },
        xAxis: {
          type: 'category',
          boundaryGap: false,
          data: this.chartData.dates
        },
        yAxis: {
          type: 'value'
        },
        series: [
          {
            name: '活跃人数',
            data: this.chartData.activeUsers,
            type: 'line',
            smooth: true,
            color: '#ff4500', // 例子颜色
          },
          {
            name: '打款金额',
            data: this.chartData.paymentAmounts,
            type: 'line',
            smooth: true,
            color: '#1e90ff', // 例子颜色
          },
          {
            name: '注册人数',
            data: this.chartData.registrationCounts,
            type: 'line',
            smooth: true,
            color: '#32cd32', // 例子颜色
          }
        ]
      };
      myChart.setOption(option);
    },

  },
}
</script>

<style>
.data-cards {
  display: flex;
  justify-content: space-around;
}

.data-card {
  flex-basis: calc(50% - 20px);
  margin: 10px;
  padding: 20px;
  box-shadow: 0 2px 4px 0 rgba(0, 0, 0, 0.1);
  border-radius: 4px;
  text-align: center;
}

.card-number {
  font-size: 2em;
  /* 数字样式 */
}
</style>
