<template>
  <div class="body-chargingStatus">
    <button @click="moveToTestAlarm">
        <img class="imglogo" src="/images/logo.png">
    </button >
    <div class="charging-container">
        <div class="charging-header">
            <h2>{{ store.chargingData.name }}</h2>
            <p>{{ store.chargingData.address }}</p>
        </div>

        <!-- 충전 진행 상황 -->
        <div class="charging-progress">
            <div class="img-container">
                <img class="img-loading" src="/images/Spinner.gif" alt="Charging Progress">
                <div class="overlay">
                    <img class="thunder" src="/images/thunder.png">
                    <span class="battery-percent">{{ store.chargingData.batteryPercent }}%</span>
                    <span class="charging-kwh">{{ store.chargingData.chargingKwh }} kWh</span>
                </div>
            </div>
            <div  v-if="store.finishAlarm.message=='충전완료'|| store.finishAlarm.message=='예약시간으로 인한 충전종료'">
                <p class="charging-status">고객님의 차량이 충전 완료되었습니다. ({{store.chargingData.carNumber}}) </p>
            </div>
            <div  v-else>
                <p class="charging-status">고객님의 차량이 충전 중 입니다. ({{store.chargingData.carNumber}}) </p>
            </div>
        </div>

        <div class="charging-info">
            <div class="info-item">
                <span>경과 시간</span>
                <strong>{{ formatElapsedTime() }}</strong>
            </div>
            <div class="info-item">
                <span>예상 남은 시간</span>
                <strong>{{ remainingTime  }}</strong>
            </div>
            <div class="info-item">
                <span>현재 요금</span>
                <strong>{{ formatCurrency(store.chargingData.amount) }}원</strong>
            </div>
            <div class="info-item">
                <span>완충 시 예상 요금</span>
                <strong>{{ formatCurrency(store.chargingData.expectAmount) }}원</strong>
            </div>
            <div class="info-item">
                <span>충전 타입</span>
                <strong>{{store.chargingData.chargerType}}</strong>
            </div>
            <div class="info-item">
                <span>단가</span>
                <strong>{{store.chargingData.pricePerKWh}}/kWh</strong>

            </div>
        </div>
        <div class="charging-actions" v-if="store.finishAlarm.message=='충전완료'|| store.finishAlarm.message=='예약시간으로 인한 충전종료'">
            <!-- <button class="start-btn" @click="connectWebSocket">충전 시작</button> -->
            <button class="stop-btn" @click="disconnectWebSocket">결제하러가기</button>
        </div>
        <div class="charging-actions" v-else>
            <!-- <button class="start-btn" @click="connectWebSocket">충전 시작</button> -->
            <button class="stop-btn" @click="disconnectWebSocket">충전 종료</button>
        </div>
    </div>
</div>  
</template>

<script setup>
import { onMounted  } from 'vue';
import { useWebSocketStore } from '@/stores/webSocketChargingStore';
import { useRouter } from 'vue-router';
import { ref } from 'vue';
import PayPrice from '../payment/PayPrice.vue';

const store = useWebSocketStore();
const router = useRouter();

const moveToTestAlarm = () => {
    router.push({
        name:'ChargingStationPage'
    }); 
};

const paymentData = ref('');


// 시간 포맷 함수
const formatDateTime = (date) => {
    if (!date) return 'N/A';
    const year = date.getFullYear();
    const month = String(date.getMonth() + 1).padStart(2, '0');
    const day = String(date.getDate()).padStart(2, '0');
    const hours = String(date.getHours()).padStart(2, '0');
    const minutes = String(date.getMinutes()).padStart(2, '0');
    const seconds = String(date.getSeconds()).padStart(2, '0');
    return `${year}-${month}-${day}T${hours}:${minutes}:${seconds}`;
};


onMounted(()=> {
    store.connectWebSocket();
})


// 종료 버튼 클릭 이벤트
const disconnectWebSocket = () => {
    store.disconnectWebSocket();
    console.log("세션 종료.")
    paymentData.value = {
            userId:store.chargingData.userId,
            stationId : store.chargingData.stationId,
            reservationId:store.chargingData.reservationId,
            amount:store.chargingData.amount,
            chargeStart:formatDateTime(store.startTime),
            chargeEnd:formatDateTime(store.endTime),
            chargingKwh:store.chargingData.chargingKwh,
            chargerId: store.chargingData.chargerId,
            name : store.chargingData.name,
            address : store.chargingData.address
        }
    alert("충전 종료 - 결제 화면으로 이동합니다.");

    router.push({
        name: 'PayPrice',
        query : {
            userId:paymentData.value.userId,
            stationId:paymentData.value.stationId,
            reservationId:paymentData.value.reservationId,
            amount:paymentData.value.amount,
            chargeStart:paymentData.value.chargeStart,
            chargeEnd:paymentData.value.chargeEnd,
            chargingKwh:paymentData.value.chargingKwh,
            chargerId:paymentData.value.chargerId,
            name:paymentData.value.name,
            address:paymentData.value.address
        }
    })
        
};

// 경과 시간을 'hh:mm:ss' 형식으로 변환하는 헬퍼 함수
const formatElapsedTime = () => {
    const hours = String(Math.floor(store.elapsedTime / 3600)).padStart(2, '0');
    const minutes = String(Math.floor((store.elapsedTime % 3600) / 60)).padStart(2, '0');
    const seconds = String(store.elapsedTime % 60).padStart(2, '0');
    return `${hours}:${minutes}:${seconds}`;
};    

const remainingTime = ref('');

watchEffect(() => {
    const remainingSeconds = store.chargingData.estimatedTimeSeconds || 0; // 예상 남은 시간 (초)
    const adjustedRemainingSeconds = remainingSeconds - store.elapsedTime; // 경과 시간 반영
    const hours = Math.floor(adjustedRemainingSeconds / 3600); // 시간 계산
    const minutes = Math.floor((adjustedRemainingSeconds % 3600) / 60); // 분 계산
    if (adjustedRemainingSeconds <= 0) {
        remainingTime.value = '완료됨';
    } else if (hours === 0) {
        remainingTime.value = `${minutes}분`;
    } else {
        remainingTime.value = `${hours}시간 ${minutes}분`;
    }
});

const formatCurrency = (amount) => {
    return Number(amount).toLocaleString('en-US');
};

watchEffect(() => {
    if (store.chargingData.batteryPercent == 100) {
        store.disconnectWebSocket(); // 웹 소켓 연결 종료

        paymentData.value = {
            userId:store.chargingData.userId,
            stationId : store.chargingData.stationId,
            reservationId:store.chargingData.reservationId,
            amount:store.chargingData.amount,
            chargeStart:formatDateTime(store.startTime),
            chargeEnd:formatDateTime(store.endTime),
            chargingKwh:store.chargingData.chargingKwh,
            chargerId: store.chargingData.chargerId,
            name : store.chargingData.name,
            address : store.chargingData.address
        }
        alert("충전 종료 - 결제 화면으로 이동합니다.");

        router.push({
            name: 'PayPrice',
            query : {
                userId:paymentData.value.userId,
                stationId:paymentData.value.stationId,
                reservationId:paymentData.value.reservationId,
                amount:paymentData.value.amount,
                chargeStart:paymentData.value.chargeStart,
                chargeEnd:paymentData.value.chargeEnd,
                chargingKwh:paymentData.value.chargingKwh,
                chargerId:paymentData.value.chargerId,
                name:paymentData.value.name,
                address:paymentData.value.address
            }
        })
    }
});


</script>

<style scoped>

@font-face {
    font-family: 'NoonnuBasicGothicRegular';
    src: url('https://fastly.jsdelivr.net/gh/projectnoonnu/noon-2410@1.0/NoonnuBasicGothicRegular.woff2') format('woff2');
    font-weight: normal;
    font-style: normal;
}

* {
    font-family: 'NoonnuBasicGothicRegular';
}
.logo img  {
    position: absolute;
    top: 5px;
    left: 5px;
    width: 150px; /* 로고 크기 조절 */
}

.thunder{
    width: 30px;
    margin-bottom: 6px;
}

.body-chargingStatus{
    display: flex;
    flex-direction: column;
    align-items: center; /* 수평 가운데 정렬 */
    justify-content: center; /* 수직 가운데 정렬 */
    background-color: #fff;
    min-height: 100vh; /* 화면 전체를 채우도록 설정 */
    padding-bottom: 0; /* 추가된 여백 제거 */
}

/* 전체 컨테이너 */
.charging-container {
    width: 650px;
    height: 830px;
    margin: auto;
    font-family: Arial, sans-serif;
    background: #ffffff;
    border-radius: 16px;
    border: 5px solid #333;
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1); 
    padding: 35px;
    text-align: center;
}

/* 충전기 정보 */
.charging-header h2 {
    font-size: 1.5em;
    color: #333;
    margin: 0;
}

.charging-header p {
    font-size: 0.9em;
    color: #555;
    margin: 5px 0;
}

/* 이미지 컨테이너 */
.img-container {
    position: relative;
    width: 500px; /* 이미지 크기 */
    height: 330px;
    margin: 20px auto;
}

.img-loading {
    width: 100%;
    height: 100%;
    border-radius: 50%; /* 원형 이미지 */
    object-fit: cover;
  
}

.overlay {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    color: #333;
    background: rgba(255, 255, 255, 0.4); /* 반투명 효과 */
    border-radius: 50%;
}

/* 배터리 퍼센트 및 충전량 */
.battery-percent {
    font-size: 1.8em;
    font-weight: bold;
}

.charging-kwh {
    font-size: 1em;
    margin-top: 5px;
}

/* 충전 상태 */
.charging-status {
    margin-top: 10px;
    font-size: 1.1em;
    font-weight: bold;
    color: #555;
}

/* 충전 상세 정보 */
.charging-info {
    margin-top: 20px;
    text-align: left;
}

.info-item {
    display: flex;
    justify-content: space-between;
    margin: 8px 0;
}

.info-item span {
    color: #555;
    font-weight: bolder;
}

.info-item strong {
    color: #333;
}

/* 충전 버튼 */
.charging-actions {
    margin-top: 45px;
    display: flex;
    justify-content: space-between;
}

.stop-btn {
    flex: 1;
    padding: 10px 15px;
    margin: 0 5px;
    border: none;
    border-radius: 8px;
    font-size: 1em;
    cursor: pointer;
    transition: background 0.3s;
    background: #333;
    color: #fff;
    height: 50px;
}



.stop-btn:hover {
    background: #676767;
}
.imglogo{
    margin: 0; /* 마진 초기화 */
    width: 150px; /* 로고 크기 조절 */
}
</style>