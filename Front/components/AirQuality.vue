<template>
  <div class="container">
    <img src="@/assets/inhatc.png" alt="로고" class="logo" />
    <!--<h1>센서 값</h1>-->
    <div class="input-group">
      <div class="sensor-average" :style="{
        color: 
          averageValue > 15 ? 'red' :
          averageValue > 9 ? 'orange' :
          averageValue > 2 ? 'green' :
          averageValue > 0.001 ? 'blue' :
          'black'
      }">
        <h4>평균 값: {{ averageValue }} {{ interpretAverageValue(averageValue) }} </h4>
        <span :style="{
          color: 
            averageValue > 15 ? 'red' :
            averageValue > 9 ? 'orange' :
            averageValue > 2 ? 'green' :
            averageValue > 0.001 ? 'blue' :
            'black'
        }">
          최근 값: {{ mostRecentValue }} {{ interpretAverageValue(mostRecentValue) }}
        </span>
      </div>
      <select v-model="selectedSensor" class="select-sensor">
        <option disabled value="null">센서 선택</option>
        <option v-for="sensor in sensorList" :key="sensor" :value="sensor">{{ sensor }}</option>
        <!-- 다른 센서 옵션을 추가 -->
      </select>
      <button @click="getData" class="get-data-button">Chart</button>
    </div>
    <div v-if="responseData" class="chart-container">
      <canvas ref="chartCanvas" class="chart-canvas"></canvas>
    </div>

    <!-- 싸이클 POST -->
    <div class="input-group">
      <label for="cycleValue">측정주기:</label>
      <input type="number" v-model="cycleValue" id="cycleValue" min="1000" />
      <button @click="changeCycle" class="change-cycle-button">변경</button>
    </div>
  </div>
</template>

<script>
import axios from 'axios';
import Chart from 'chart.js/auto';

export default {
  data() {
    return {
      selectedSensor: null,
      responseData: null,
      chart: null,
      cycleValue: 3000,
      sensorList: [],
      averageValue: 0 ,
      mostRecentValue: 0
    };
  },
  methods: {
    async getSensorList() {
      try {
        const response = await axios.get('https://port-0-raspberry-pi-project-5mk12alpbcv53c.sel5.cloudtype.app/sensors');
        this.sensorList = response.data;
      } catch (error) {
        console.error('Error fetching sensor list:', error);
      }
    },
    async getData() {
      try {
        const response = await axios.get(`https://port-0-raspberry-pi-project-5mk12alpbcv53c.sel5.cloudtype.app/sensors/graph/${this.selectedSensor}`);
        this.responseData = response.data;
        
        const average = this.calculateAverage(); // 평균값 계산
        this.renderChart(average); // 차트 렌더링에 평균값 전달
        if (this.responseData.length > 0) {
          this.mostRecentValue = this.responseData[0].measure_value;
        } else {
          this.mostRecentValue = 0; // 데이터가 없는 경우 기본값 설정 혹은 필요에 따라 처리
        }
              
      } catch (error) {
        console.error('Error fetching data:', error);
      }
    },
    calculateAverage() {
      const values = this.responseData.map(item => item.measure_value);
      const sum = values.reduce((accumulator, currentValue) => accumulator + currentValue, 0);
      const average = sum / values.length;
      const roundedAverage = average.toFixed(2);
      console.log(`선택된 센서에 대한 값의 평균: ${roundedAverage}`);
      this.averageValue = roundedAverage;
      return roundedAverage; // 평균값 반환
    },
    interpretAverageValue(averageValue) {
    if (averageValue >= 15) {
      return '(매우나쁨)';
    } else if (averageValue >= 9) {
      return '(나쁨)';
    } else if (averageValue >= 2) {
      return '(보통)';
    } else if (averageValue >= 0.001) {
      return '(좋음)';
    }
  },
    renderChart(average) {
  if (this.chart) {
    this.chart.destroy(); // 기존 차트 제거
  }
  const labels = [];
  const values = [];

  // 센서 데이터를 내림차순으로 정렬
  this.responseData.sort((a, b) => new Date(b.measure_time) - new Date(a.measure_time));

  this.responseData.forEach(item => {
    const date = new Date(item.measure_time);
    const month = date.getMonth() + 1;
    const day = date.getDate();
    const hours = date.getHours();
    const minutes = date.getMinutes();
    const seconds = date.getSeconds();
    
    labels.push(`${month}-${day} ${hours}:${minutes}:${seconds}`);
    values.push(item.measure_value);
  });

  const ctx = this.$refs.chartCanvas.getContext('2d');
  this.chart = new Chart(ctx, {
    type: 'line',
    data: {
      labels: labels.reverse(), // labels 배열을 역순으로 변경하여 최신 데이터를 오른쪽에 표시
      datasets: [
        {
          label: `${this.selectedSensor} Values`,
          data: values.reverse(), // values 배열도 역순으로 변경
          backgroundColor: 'rgba(75, 192, 192, 1)',
          borderColor: 'rgba(75, 192, 192, 1)',
          borderWidth: 3
        },
        {
          label: 'Average',
          data: Array(values.length).fill(average),
          borderColor: 'rgba(0, 0, 200, 0.3)',
          borderWidth: 1,
          borderDash: [5, 3],
          fill: false
        }
      ]
    },
    options: {
      responsive: true,
      maintainAspectRatio: false
    }
  });
},

    async changeCycle() {
      try {
        const requestBody = {
          name: this.selectedSensor,
          cycle: this.cycleValue
        };

        const response = await axios.post('https://port-0-raspberry-pi-project-5mk12alpbcv53c.sel5.cloudtype.app/change', requestBody);
        console.log('POST 요청 성공:', response.data);

      } catch (error) {
        console.error('POST 요청 실패:', error);
      }
    }
  },
  mounted() {
    this.getSensorList();
    this.getData();
  }
};
</script>

<style>
.container {
  max-width: 1000px;
  margin: 150px auto 0; /* 상단 여백을 조정하여 컨테이너를 상단에서 20px 떨어지게 설정 */
  padding: 20px;
  font-family: 'Arial', sans-serif;
  background-color: #f5f5f5;
  border-radius: 8px;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
}


h1 {
  text-align: center;
  margin-bottom: 20px;
  color: #333;
}

.input-group {
  display: flex;
  align-items: center;
  justify-content: flex-end;
}

.label {
  margin-right: 10px; /* 라벨과 인풋 사이의 간격 조절 */
}
#cycleValue {
  text-align: right;
}
.select-sensor {
  flex: 0 0 auto; /* 유연한 크기 조정을 막고 고정 크기로 설정 */
  width: 150px; /* 원하는 크기로 설정 */
  padding: 8px; /* 적절한 패딩 적용 */
  font-size: 14px;
  border: 1px solid #ccc;
  border-radius: 4px;
  margin-left: auto; /* 오른쪽으로 배치 */
  margin-right: 10px; /* 오른쪽 여백 설정 */
  outline: none;
}
.logo {
  width: 200px; /* 로고 이미지의 너비 설정 */
  height: auto; /* 높이 자동 조정 */
  position: absolute; /* 위치를 설정하기 위해 절대 위치 지정 */
  top: 30px; /* 컨테이너 상단에서의 위치 조정 */
  left: 30px; /* 컨테이너 왼쪽에서의 위치 조정 */
}

.get-data-button, .change-cycle-button {
  padding: 10px 20px;
  border: none;
  border-radius: 4px;
  background-color: #4caf50;
  color: #fff;
  cursor: pointer;
  transition: background-color 0.3s;
}
.get-data-button{
  font-size: 12px;
  padding: 5px 10px;
}

.get-data-button:hover, .change-cycle-button {
  background-color: #45a049;
}

.chart-container {
  border: 1px solid #ddd;
  padding: 20px;
  border-radius: 4px;
  background-color: #fff;
  display: flex; /* 부모 요소를 플렉스로 설정하여 자식 요소를 정렬 */
  justify-content: center; /* 가로 방향 가운데 정렬 */
  align-items: center; /* 세로 방향 가운데 정렬 */
  margin-top: 20px; /* 상단 여백을 추가하여 차트를 조금 더 아래쪽에 배치 */
}
.chart-canvas {
  width: 100%; /* 차트 요소를 부모 요소의 100%로 설정 */
  height: 400px; /* 원하는 높이로 설정하세요 */
}
.cycle-input {
  display: flex;
  justify-content: center;
  align-items: center;
}

.cycle-input label {
  margin-right: 10px;
}

#cycleValue {
  padding: 8px;
  font-size: 14px;
  border: 1px solid #ccc;
  border-radius: 4px;
  width: 100px;
  margin-right: 10px;
}

.change-cycle-button {
  padding: 10px 18px;
  background-color: #4285f4;
}
.sensor-average{
  margin-left: 10px;
  font-size:15px;
}
/* ... */
</style>
