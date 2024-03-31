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
      <h1 style="align-items: center;text-align: center;margin: 0;">近七日注册人数变化</h1>
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
    return {
      cards: [
        { label: '注册总人数', number: 0 },
        { label: '今日注册人数', number: 0 },
      ],
      chartData: {
        dates: [],
        counts: [],
      },
    }
  },

  mounted() {
    this.fetchUserData();
    this.fetchRegistrationData();
  },

  methods: {
    fetchUserData() {
      const todayStart = new Date();
      todayStart.setHours(0, 0, 0, 0);

      const todayEnd = new Date();
      todayEnd.setHours(23, 59, 59, 999);

      db.collection('user')
        .get()
        .then(res => {
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

    fetchRegistrationData() {
      // 获取近七天的日期和每天的注册数
      for (let i = 6; i >= 0; i--) {
        const date = new Date();
        date.setDate(date.getDate() - i);
        const dateString = date.toISOString().split('T')[0];

        this.chartData.dates.push(dateString);
        this.chartData.counts.push(0); // 初始设置为0
      }

      db.collection('user')
        .get()
        .then(res => {
          res.data.forEach(user => {
            console.log(user)
            // 确保将createTime正确解析为日期对象并格式化为YYYY-MM-DD
            const createTime = new Date(user.data.createTime).toISOString().split('T')[0];
            const index = this.chartData.dates.indexOf(createTime);
            if (index !== -1) {
              this.chartData.counts[index]++;
            }
          });

          this.drawRegistrationChart();
        })
        .catch(err => {
          console.error('Error getting registration data:', err);
        });
    },

    drawRegistrationChart() {
      const chartDom = this.$refs.registrationChart;
      const myChart = echarts.init(chartDom);
      const option = {
        // title: {
        //   text: '近七天注册人数变化'
        // },
        tooltip: {
          trigger: 'axis'
        },
        xAxis: {
          type: 'category',
          boundaryGap: false,
          data: this.chartData.dates
        },
        yAxis: {
          type: 'value'
        },
        series: [{
          data: this.chartData.counts,
          type: 'line',
          areaStyle: {}
        }]
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

.card-content {
  /* 内容样式 */
}

.card-number {
  font-size: 2em;
  /* 数字样式 */
}

.card-label {
  /* 标签样式 */
}
</style>
