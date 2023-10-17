<template>
  <div class="row g-0">
    <div class="col-3 overflow-scroll" style="max-height: 100vh;">
      <div class="" >
        <div class="d-flex align-items-center py-2 px-2">
          <div class="me-2">
            <label for="cityName">縣市:</label>
          </div>
          <div class="flex-grow-1">
            <select name="" id="cityName" class="form-control" v-model="city">
              <option :value="item.CityName" v-for="(item, i) in cityName" :key="i">{{ item.CityName }}</option>
            </select>
          </div>
        </div>

        <div class="d-flex align-items-center py-2 px-2">
          <div class="me-2">
            <label for="Region">地區:</label>
          </div>
          <div class="flex-grow-1">
            <select name="" id="Region" class="form-control" v-model="region">
              <option :value="item.AreaName" v-for="item in regionName">{{ item.AreaName }}</option>
            </select>
          </div>
        </div>
      </div>

      <div class="">
        <div v-for="item in data" class="border px-2 pt-2 ">
          <h3>藥局名稱: {{ item.properties.name }}</h3>
          <p>成人口罩:{{ item.properties.mask_adult }} | 兒童口罩:{{ item.properties.mask_child }}</p>
          <p>地址:{{ item.properties.address }}</p>
        </div>
      </div>

    </div>
    <div class="col-9">
      <div id="map"></div>
    </div>
  </div>
</template>

<script setup>
import 'leaflet/dist/leaflet.css';
import L from 'leaflet';
import axios from 'axios';
import { onMounted, ref, watch } from 'vue';
let data = ref([]);
let cityName = ref([]);
let regionName = ref([]);
let city = ref("臺北市");
let region = ref("中正區");

//必須在全域定義
let map = null;

//監聽縣市 回傳對應的鄉鎮區
watch(city, (newVal, oldVal) => {
  regionName.value = cityName.value.filter((item) => {
    //回傳相對應的縣市如:臺北市 == 臺北市
    return item.CityName == newVal;
  });
  //將鄉鎮區再次賦予
  regionName.value = regionName.value[0].AreaList;
  region.value = regionName.value[0].AreaName;
});

//重撈資料 並且篩選
watch(region, (newVal, oldVal) => {
  // console.log(newVal);
  //每次觸發前都先刪除先前打的點
  map.eachLayer((layer) => {
    if (layer instanceof L.Marker) {
      map.removeLayer(layer);
    }
  });
  getData().then(() => {
    data.value = data.value.filter((item) => {
      return item.properties.town == newVal && item.properties.county == city.value;
    });
  });
})


onMounted(() => {
  //如果將let map寫在該區域會有問題
  map = L.map('map').setView([25.032361, 121.518267], 10);
  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    attribuztion: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors'
  }).addTo(map);
  getData();
  getRegionData();

})

const getData = () => {
  return axios.get('https://raw.githubusercontent.com/kiang/pharmacies/master/json/points.json')
    .then((res) => {
      data.value = res.data.features;
      data.value = data.value.filter((item) => {
        return item.properties.town == region.value && item.properties.county == city.value;
      });

      for (let i = 0; i < data.value.length; i++) {
        //將經緯度單獨拆出來
        const coordinates = [data.value[i].geometry.coordinates[1], data.value[i].geometry.coordinates[0]];
        const maker = L.marker(coordinates).addTo(map)
          .bindPopup(`<div><h5>藥局名稱 :${data.value[i].properties.name} </h5><p class="py-1 m-0">成人口罩: ${data.value[i].properties.mask_adult}</p><p class="py-1 m-0">小孩口罩: ${data.value[i].properties.mask_child}</p><div>`)
          if(i == 0){
            maker.openPopup();
            //將定位點定於撈取到的資料的第一筆經緯度 位置就可以正確
            map.setView(coordinates, 16);
          }else{
            maker.closePopup();
          }
      }
    })
}

const getRegionData = () => {
  axios.get("https://raw.githubusercontent.com/donma/TaiwanAddressCityAreaRoadChineseEnglishJSON/master/AllData.json")
    .then((res) => {
      cityName.value = res.data;
      // console.log(cityName.value);
      regionName.value = cityName.value.filter((item) => {
        //回傳相對應的縣市如:臺北市 == 臺北市
        return item.CityName == "臺北市";
      });
      //將鄉鎮區再次賦予
      regionName.value = regionName.value[0].AreaList;
    })
}
</script>

<style>
.row {
  height: 100% !important;
  width: 100% !important;
}

#map {
  height: 100vh;
}

.overflow-scroll {
  overflow-x: hidden !important;
}
</style>
