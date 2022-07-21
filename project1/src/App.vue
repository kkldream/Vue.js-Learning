<template>
  <div id="app">
    <button @click="getLocation()">getLocation</button>
    <button @click="updateBikeStationMap()">updateBikeStationMap</button>
    <br>
    <div id="map"></div>
  </div>
</template>

<script>
import L from 'leaflet';
import img from './assets/here.png';
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
    tdxToken: null,
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
      var myIcon = L.icon({
        iconUrl: img,
        iconSize: [41, 49],
        iconAnchor: [20, 49],
        popupAnchor: [0, -50],
      });
      this.marker.myPos.point = L.marker(pos, { icon: myIcon }).addTo(openStreetMap);
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

      // 取得TDX金鑰
      async function getTdxTokenKey() {
        // 取得API資料
        const apiUrl = 'https://tdx.transportdata.tw/auth/realms/TDXConnect/protocol/openid-connect/token';
        let headers = {
          'Content-Type': 'application/x-www-form-urlencoded;charset=UTF-8',
        }
        let details = {
          grant_type: "client_credentials",
          client_id: "b0742033-ec2e35c5-abbc-4252",
          client_secret: "6e63241d-a57f-405e-9ce9-ebaab604ea6f",
        }
        var formBody = [];
        for (var property in details) {
          var encodedKey = encodeURIComponent(property);
          var encodedValue = encodeURIComponent(details[property]);
          formBody.push(encodedKey + "=" + encodedValue);
        }
        formBody = formBody.join("&");
        let response = await fetch(apiUrl, { method: "POST", headers: headers, body: formBody });
        let response_json = await response.json();
        return response_json;
      }

      // 取得腳踏車即時資訊
      async function getAvailableBikeInfo() {
        let key = await getTdxTokenKey();
        // 取得API資料
        const apiUrl = 'https://tdx.transportdata.tw/api/basic/v2/Bike/Availability/City/Taipei?%24format=JSON';
        let headers = {
          "Content-Type": "application/json",
          "Accept": "application/json",
          "authorization": "Bearer " + key.access_token,
        }
        let response = await fetch(apiUrl, { method: "GET", headers: headers });
        let response_json = response.json();
        return response_json
      }

      // 更新標記腳踏車站標記點
      async function getBikeStationMap(data) {
        // 取得API資料
        const apiUrl = 'https://localhost:44330/api/bike/station';
        let headers = {
          "Content-Type": "application/json",
          "Accept": "application/json",
        }
        let body = {
          "lng": data.myPos.lng,
          "lat": data.myPos.lat,
          "range": 1000,
        }
        let response = await fetch(apiUrl, { method: "POST", headers: headers, body: JSON.stringify(body) });
        let response_json = await response.json();
        return response_json;
      }

      let availableBikeInfo;
      getAvailableBikeInfo()
        .then(available => {
          availableBikeInfo = available;
          return getBikeStationMap(this);
        })
        .then(station => {
          station.forEach(sta => {
            const info = availableBikeInfo.find(obj => obj.StationUID == sta.StationUID)
            sta.AvailableRentBikes = info.AvailableRentBikes;
            sta.AvailableReturnBikes = info.AvailableReturnBikes;
            sta.UpdateTime = info.UpdateTime;
          })
          console.log(station);
          return station;
        })
        // 將資料標記在地圖上
        .then(station => {
          // 先清除舊標記
          this.marker.bikeStation.forEach(r => r.remove())
          this.marker.bikeStation = [];
          // 添加新標記
          station.forEach(sta => {
            let marker = L.marker([sta.StationPosition[1], sta.StationPosition[0]]).addTo(openStreetMap).bindPopup(
              `<p><strong style="font-size: 20px;">${sta.StationName}</strong></p>
              <strong style="font-size: 16px; color: #d45345;">${sta.StationAddress}</strong><br>
              直線距離: ${parseInt(sta.Distance)}公尺<br>
              可租借數量: ${sta.AvailableRentBikes}  可歸還數量: ${sta.AvailableReturnBikes}<br>`
            );
            this.marker.bikeStation.push(marker);
          });
        })
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

    () => { // 取得TDX金鑰
      const apiUrl = 'https://tdx.transportdata.tw/auth/realms/TDXConnect/protocol/openid-connect/token';
      let headers = {
        'Content-Type': 'application/x-www-form-urlencoded;charset=UTF-8',
      }
      let body = {
        grant_type: "client_credentials",
        client_id: "b0742033-ec2e35c5-abbc-4252",
        client_secret: "6e63241d-a57f-405e-9ce9-ebaab604ea6f",
      }
      var formBody = [];
      for (var property in body) {
        var encodedKey = encodeURIComponent(property);
        var encodedValue = encodeURIComponent(body[property]);
        formBody.push(encodedKey + "=" + encodedValue);
      }
      formBody = formBody.join("&");
      fetch(apiUrl, { method: "POST", headers: headers, body: formBody })
        .then(res => res.json())
        .then(res => {
          console.log(res);
          this.tdxToken = res.access_token;
        });
    }
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