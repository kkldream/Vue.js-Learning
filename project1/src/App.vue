<template>
  <div id="app">
    <button @click="getLocation()">定位</button>
    <button @click="updateBikeStationMap()">PTX</button>
    <br>
    <div id="map"></div>
  </div>
</template>

<script>
import L from 'leaflet';
let openStreetMap = {};
export default {
  name: 'App',
  data: () => ({
    myPos: {
      lat: 25.049245,
      lng: 121.515494,
      radius: 1,
    },
    marker: {
      myPos: {
        point: null,
        circle: null,
      },
      bikeStation: [],
    },
  }),
  components: {

  },
  computed: {

  },
  watch: {

  },
  methods: {

    // 取得定位資訊
    getLocation() {
      if (navigator.geolocation) {
        navigator.geolocation.getCurrentPosition((p) => {
          console.log(p);
          this.myPos.lat = p.coords.latitude
          this.myPos.lng = p.coords.longitude;
          this.myPos.radius = p.coords.accuracy;
          this.markerMyPoint(); // 呼叫標記動作
        });
      }
    },

    // 標記定位點
    markerMyPoint() {
      let pos = [this.myPos.lat, this.myPos.lng];
      let radius = this.myPos.radius;
      // 先將原本標記點清除
      if (this.marker.myPos.point) this.marker.myPos.point.remove();
      if (this.marker.myPos.circle) this.marker.myPos.circle.remove();
      // 建立中心標記點
      this.marker.myPos.point = L.marker(pos).addTo(openStreetMap);
      this.marker.myPos.point.bindPopup("I am here.");
      // 建立圓形區塊
      this.marker.myPos.circle = L.circle(pos, {
        color: 'red',
        fillColor: '#f03',
        fillOpacity: 0.1,
        radius: radius
      }).addTo(openStreetMap);
      // 移動地圖至定位位置
      openStreetMap.flyTo(pos, 18);
    },

    // 更新標記腳踏車站標記點
    updateBikeStationMap() {
      // 取得API資料
      const apiUrl = 'https://localhost:44330/api/bike/station';
      let headers = {
        "Content-Type": "application/json",
        "Accept": "application/json",
      }
      let body = {
        "lng": this.myPos.lng,
        "lat": this.myPos.lat,
        "range": 1000,
      }
      fetch(apiUrl, { method: "POST", headers: headers, body: JSON.stringify(body) })
        .then(res => res.json())
        // 將資料標記在地圖上
        .then(res => {
          // 先清除舊標記
          this.marker.bikeStation.forEach(r => r.remove())
          this.marker.bikeStation = [];
          // 添加新標記
          res.forEach(r => {
            let marker = L.marker([r.Position[1], r.Position[0]]).addTo(openStreetMap).bindPopup(
              `<p><strong style="font-size: 20px;">${r.StationUID}</strong></p>
              <strong style="font-size: 16px; color: #d45345;">Youbkie ${r.ServiceType}.0</strong><br>
              直線距離: ${parseInt(r.Distance)}<br>
              剩餘數量: ${r.BikesCapacity}<br>`
            );
            this.marker.bikeStation.push(marker);
          });
        });
    },
  },

  // 網頁載入時執行
  mounted() {
    openStreetMap = L.map('map', {
      center: [this.myPos.lat, this.myPos.lng],
      zoom: 12,
    });
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
      maxZoom: 19,
      attribution: '© OpenStreetMap'
    }).addTo(openStreetMap);
  },
};
</script>

<style lang="scss">
@import 'bootstrap/scss/bootstrap';

#map {
  position: relative;
  height: 100vh;
}
</style>