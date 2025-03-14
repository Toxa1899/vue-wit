<script setup>
import SlaidBar from "../components/SlaidBar.vue";
import { ref, onMounted, computed } from "vue";
import L from "leaflet";
import "leaflet/dist/leaflet.css";
import "leaflet.markercluster"; // Импортируем кластеризацию маркеров
import axios from "axios";

const map = ref(null);
const userMarkers = ref(new Map()); // Хранение маркеров для каждого пользователя
const markersCluster = ref(null); // Кластер для всех маркеров
const userPolylines = ref(new Map());
const users = ref([]); // Список пользователей
const searchQuery = ref(""); // Запрос поиска

// Инициализация карты
const initMap = () => {
  map.value = L.map("map", {
    center: [34.0522, -118.2437],
    zoom: 14,
    minZoom: 3,
    maxZoom: 18,
    scrollWheelZoom: true,
    dragging: true,
  });

  // Добавляем слой карты
  L.tileLayer("https://{s}.basemaps.cartocdn.com/dark_all/{z}/{x}/{y}{r}.png", {
    maxZoom: 18,
  }).addTo(map.value);

  // Инициализация кластера маркеров
  markersCluster.value = L.markerClusterGroup({
    iconCreateFunction: (cluster) => {
      const count = cluster.getChildCount(); // Получаем количество маркеров в кластере
      return L.divIcon({
        html: `<div style="color: yellow; font-size: 16px; font-weight: bold;">${count}</div>`, // Устанавливаем желтый цвет и жирный текст
        className: "custom-cluster-icon", // Класс для дальнейшей кастомизации
        iconSize: L.point(40, 40) // Размер иконки
      });
    }
  });
  map.value.addLayer(markersCluster.value);
};

// Функция для создания или обновления маркера для пользователя
const addUserMarker = (user) => {
  const newLocation = [user.latitude, user.longitude];
  
  // Проверяем, существует ли маркер для данного пользователя
  let userMarker = userMarkers.value.get(user.id);
  let userPolyline = userPolylines.value.get(user.id);

  if (userMarker) {
    // Если маркер уже существует, обновляем его положение
    userMarker.setLatLng(newLocation);
    userPolyline.addLatLng(newLocation);
  } else {
    // Если маркер не существует, создаем новый
    userMarker = L.marker(newLocation).bindPopup(`<b>Driver: ${user.user}</b>`);
    markersCluster.value.addLayer(userMarker); // Добавляем маркер в кластер
    
    // Создаем полилинию для отслеживания пути
    userPolyline = L.polyline([newLocation], { color: "silver", weight: 2, opacity: 0.6 });
    userPolyline.addTo(map.value);
    
    // Сохраняем маркер и полилинию в соответствующие мапы
    userMarkers.value.set(user.id, userMarker);
    userPolylines.value.set(user.id, userPolyline);
  }
};

// Функция для получения данных пользователей с сервера
const fetchUserLocations = async () => {
  try {
    const accessToken = localStorage.getItem("accessToken");
    const response = await axios.get(
      "http://34.40.8.20/api/v1/map/locations/",
      {
        headers: { Authorization: `Bearer ${accessToken}` },
      }
    );
    users.value = response.data.results;

    // Добавляем маркеры для всех пользователей
    users.value.forEach((user) => {
      addUserMarker(user);
    });
  } catch (error) {
    console.error("Ошибка при получении данных от бэкенда:", error);
  }
};

// Фильтрация пользователей на основе поискового запроса
const filteredUsers = computed(() => {
  return users.value.filter((user) => {
    return String(user.user).toLowerCase().includes(searchQuery.value.toLowerCase());
  });
});

// Функция для зума на пользователя при выборе из списка
const zoomToUser = (user) => {
  const marker = userMarkers.value.get(user.id);
  if (marker) {
    map.value.setView(marker.getLatLng(), 15); // Устанавливаем зум и центруем карту на пользователя
    marker.openPopup(); // Открываем попап маркера
  }
};

// Инициализация карты и запуск интервала для обновления данных
onMounted(() => {
  initMap();

  // Получаем данные о пользователях
  fetchUserLocations();

  // Обновляем данные каждые 5 секунд
  setInterval(() => {
    fetchUserLocations();
  }, 5000);
});
</script>
<template>
  <div class="maps">
    <div id="map" style="height: 800px; width: 100%"></div>
    <SlaidBar />

    <!-- САЙДБАР -->
    <div class="main">
      <header class="main-header">
        <div class="main-header__content">
          <div class="main-btn">
            <div class="main-dash">
              <button class="main-btn-logs">Dashboard</button>
              <button class="main-btn-logs">Logs</button>
              <button class="main-btn-logs">Logs by driver</button>
              <button class="main-btn-logs">Trackings</button>
            </div>
            <input class="main-input" type="search" placeholder="Search..." />
            <div class="main-inputs-btn">
              <button class="main-btn-in">
                <i class="fas fa-search"></i>
              </button>
              <button class="main-btn-in">
                <i class="fas fa-times"></i>
              </button>
            </div>
          </div>
        </div>
      </header>
      <div class="box">
        <input
          class="box-input"
          type="search"
          v-model="searchQuery"
          placeholder="search by name"
        />
        <div
          class="box-border"
          v-for="user in filteredUsers"
          :key="user.id"
          @click="zoomToUser(user)"
        >
          <div class="box-small">
            <h4 class="box-h4">{{ user.user }}</h4>
            <div class="box-mph">53 mph</div>
          </div>
          <div class="box-location">
            <img src="../icons/Icon - location.svg" alt="" />
            <p>3.43 mi S of Mayview, MO</p>
            <img class="box-copy" src="../icons/icon - copy.svg" alt="" />
          </div>
          <div class="box-small-p">
            <p class="box-p">Started: 08-02-2024, 20:36 CDT</p>
          </div>
        </div>
        <!-- <div  class="section" @click="toggleSection2()">
          <div class="section-title">Diagnostics</div>
          <img src="../icons/Icon - arrow.svg" alt="" />
        </div>
        <div id="section-content2" class="section-content">
          <p>Diagnostic content goes here...</p>
        </div>

        <div  class="section" @click="toggleSection()">
          <div class="section-title">Trip history</div>
          <img src="../icons/Icon - arrow.svg" alt="" />
        </div>
        <div id="section-content" class="section-content">
          <p>Trip history content goes here...</p>
        </div> -->
      </div>
    </div>
  </div>
</template>
  

  <style scoped>
@import url("https://fonts.googleapis.com/css2?family=Inter+Tight:ital,wght@0,100..900;1,100..900&family=Inter:ital,opsz,wght@0,14..32,100..900;1,14..32,100..900&display=swap");
.leaflet-control-attribution {
  opacity: 0;
}
.sidebar {
  z-index: 1;
}
.maps {
  height: 100%;
  margin: 0;
  position: relative;
}
#map {
  height: 100%; /* Карта занимает весь контейнер */
  width: 100%;
  background-color: #1b1b1b;
  z-index: 0; /* Устанавливаем низкий z-index для карты */
}
.menu-item {
  cursor: pointer;
  display: flex;
  align-items: center;
}

.menu-item img {
  margin-right: 10px;
}

.active img {
  filter: hue-rotate(0deg) brightness(1) contrast(1) saturate(1) sepia(0)
    drop-shadow(0 0 0) !important; /* Или вы можете использовать другой способ изменения цвета */
}

.active-text {
  color: #df4924; /* Цвет текста при активном состоянии */
}

/* Стили для кнопки */
.sidebar-btn {
  cursor: pointer;
}
/* Базовые стили для свернутой боковой панели */
.sidebar {
  position: fixed;
  height: 100%;
  width: 100px; /* Ширина боковой панели по умолчанию */
  top: 0;
  left: 0;
  background-color: #111;
  display: flex;
  flex-direction: column;

  /* padding-top: 10px; */
  transition: width 0.3s; /* Плавная анимация изменения ширины */
}

/* Развернутая боковая панель */
.sidebar.expanded {
  width: 300px; /* Ширина при развернутой панели */
}

/* Стили для текста в свернутом состоянии */
.sidebar p span {
  display: none; /* Прячем текст по умолчанию */
}

/* Когда боковая панель развернута, показываем текст */
.sidebar.expanded p span {
  display: inline-block;
  margin-left: 10px;
  color: white;
  font-size: 18px;
  font-family: "inter"; /* Добавляем отступ слева */
}

/* Стрелка для открытия/закрытия панели */
#toggle-btn {
  position: absolute;
  top: 20px;
  right: 10px; /* Положение стрелки */
  cursor: pointer;
}

/* Поворот стрелки при развернутой панели */
.sidebar.expanded #toggle-btn img {
  transform: rotate(180deg);
}

.sidebar-arrow {
  /* padding-right: 550px; */
  position: relative;
  left: 50px;
  top: 10px;
}

/* Main content style */
.main {
  margin-left: 120px;
  max-width: 1520px;
  margin: 0 auto;
  padding: 16px;
  position: absolute;
  left: 10%;
  top: 0;
}

.sterl-right {
  display: flex;
  padding-left: 57px;
}
.sidebar-btn {
  background-color: transparent;
  border: none;
}
.main-btn-logs {
  width: 146px;
  height: 34px;
  border-radius: 50px;
  background-color: #3e3d3e;
  border: none;
  color: white;
  font-family: "inter";
  font-size: 16px;
  font-weight: 400;
  line-height: 33px;
}
.main-btn-logs:nth-child(1) {
  background-color: #df4924;
  color: black;
}
.main-btn-in {
  width: 146px;
  height: 34px;
  border-radius: 50px;
  background-color: #3e3d3e;
  border: none;
  color: white;
  font-family: "inter";
  font-size: 16px;
  font-weight: 400;
  line-height: 33px;
}
.main-btn {
  margin-top: 20px;
  display: flex;
  gap: 30px;
}
.main-dash {
  display: flex;
  gap: 30px;
}

/* Стили для поля ввода */
.main-input {
  width: 200px; /* Немного увеличим ширину */
  height: 34px;
  border-radius: 50px;
  background-color: #3e3d3e;
  border: none;
  padding-left: 15px; /* Внутренний отступ для текста */
  color: white; /* Убираем стандартное обрамление */
  margin-right: 10px;
  font-size: 14px;
}
p {
  margin-top: 40px;
}
.main-input::placeholder {
  color: #a3a3a3; /* Цвет плейсхолдера */
}

/* Стили для кнопок */
.main-inputs-btn {
  display: flex;
  gap: 10px; /* Пространство между кнопками */
}

.main-btn-in {
  width: 134px;
  height: 34px;
  /* border-radius: 50%; */
  background-color: #575757; /* Цвет кнопки */
  border: none;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  color: white;
  font-size: 16px;
  transition: background-color 0.3s ease;
}

.main-btn-in:hover {
  background-color: #ff9900; /* Цвет кнопки при наведении */
}

/* Иконки FontAwesome внутри кнопок */
.main-btn-in i {
  color: white;
}

.box {
  position: absolute;
  width: 400px;
  height: 594px;
  top: 133px;
  background-color: #3e3d3e;
  border-radius: 25px;
  left: 90%;
}
.box-input {
  width: 373px;
  height: 41px;
  border-radius: 20px;
  background: transparent;
  border-color: white;
  margin: 20px 13px;
  font-size: 16px;
  text-align: end;
  color: white;
}
.box-input:focus {
  width: 373px;
  transition: width 0.3s ease;
}
.box-small {
  display: flex;
  gap: 20px;
  justify-content: space-evenly;
  /* border-bottom: 2px solid black; */
}
.box-h4 {
  font-family: "inter";
  font-size: 16px;
  font-weight: 400;
  color: white;
  margin: 0;
}
.box-mph {
  margin-top: 12px;
  padding: 7px 7px;
  width: 61px;
  height: 20px;
  border-radius: 10px;
  font-family: "inter";
  font-size: 16px;
  font-weight: 400;
  line-height: 15px;
  color: white;
  background-color: #df4924;
  border: none;
}
.box-location {
  display: flex;
  font-family: "inter";
  font-size: 16px;
  font-weight: 200;
  line-height: 13px;
  color: white;
  padding-left: 25px;
}
.box-location p {
  margin: 0;
}
.box-copy {
  padding-left: 5px;
}
.box-p {
  padding-bottom: 20px;
  font-family: "inter";
  padding-left: 25px;
  color: white;
  font-weight: 100;
  line-height: 1px;
  font-size: 16px;
  border-bottom: 2px solid gray;
  margin-top: 10px;
}
.box-border {
  cursor: pointer;
}
.box-title {
  font-family: "inter";
  font-size: 14px;
  font-weight: 400;
  line-height: 13px;
  text-align: left;
  border: 2px solid black;
}
.section {
  border-bottom: 2px solid gray;
  padding: 15px 27px;
  margin: 10px 0;
  /* border-radius: 5px; */
  display: flex;
  justify-content: space-between;
  align-items: center;
  cursor: pointer;
  font-family: "inter";
  color: white;
}
.section-title {
  font-size: 18px;
}
.arrow {
  font-size: 18px;
  color: #ff4500;
  transform: rotate(0deg);
  transition: transform 0.3s ease;
}
.expanded .arrow {
  transform: rotate(180deg);
}
.section-content {
  display: none;
  padding: 10px 0;
  color: #ccc;
}
.expandeds {
  display: block;
  padding: 10px 50px;
}
.sidebar p {
  align-items: center;
  padding-left: 30px;
}
@media screen and (max-width: 1690px) {
  .main {
    left: 5%;
  }
  @media screen and (max-width: 1615px) {
    .main {
      left: 0%;
    }
  }
  @media screen and (max-width: 1538px) {
    .main-inputs-btn {
      position: absolute;
      top: 90px;
      left: 161px;
    }
    .main-input {
      position: relative;
      top: 50px;
      right: 200px;
    }
    .main {
      left: 25%;
    }
    .box {
      left: 90%;
      top: 170px;
    }
    @media screen and (max-width: 921px) {
      /* Боковая панель становится шире или уже */
      .sidebar {
        width: 60px; /* Можете изменить на другое значение */
      }

      /* Когда боковая панель развернута */
      .sidebar.expanded {
        width: 180px;
      }

      /* Убираем текст, если не помещается */
      .sidebar p {
        font-size: 12px; /* Уменьшаем текст */
      }

      .main {
        margin-left: 35%; /* Подстраиваем основной контент */
      }

      /* Настройка кнопок для мобильных */
      .main-btn-logs,
      .main-btn-in {
        width: 100px;
        height: 28px;
        font-size: 14px;
      }

      /* Поле ввода на мобильных */
      .main-input {
        width: 150px;
      }
      .body {
        background-repeat: no-repeat;
      }

      .main-input:focus {
        width: 180px;
      }
      .sidebar {
        height: 989px;
      }
      .main-dash {
        gap: 10;
      }
      .box {
        left: 15%;
        top: 170px;
      }
      .main-btn {
        margin-left: 30px;
      }
      .main-inputs-btn {
        position: absolute;
        top: 90px;
        left: 161px;
      }
      .main-input {
        position: relative;
        top: 50px;
        right: 200px;
      }
      .fas1 {
        margin-top: 90px;
      }
      p {
        margin-top: 40px;
      }
      
    }
  }
}
</style>
  
  
  