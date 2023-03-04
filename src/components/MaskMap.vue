<template>
  <v-container>
    <v-row>
      <v-col cols="12" sm="3">
        <v-autocomplete
          v-model="selected_county"
          :items="county_town_data"
          label="CITY"
          outlined
          chips
          item-text="CityName"
          return-object
          @change="showTown"
        >
        </v-autocomplete>

        <v-autocomplete
          v-model="selected_town"
          :items="town_list"
          label="COUNTRY"
          outlined
          chips
          item-text="AreaName"
          return-object
          @change="updateMaskData"
        >
          <!-- @change="getMaskData" -->
        </v-autocomplete>
      </v-col>

      <v-col cols="12" sm="9">
        <!-- map -->
        <div id="app">
          <!-- 初始化地圖設定 -->
          <l-map
            ref="myMap"
            :zoom="zoom"
            :center="center"
            :options="options"
            style="height: 90vh"
          >
            <!-- 載入圖資 -->
            <l-tile-layer :url="url" :attribution="attribution" />

            <!-- 自己所在位置 -->
            <l-marker ref="location" :lat-lng="your_location">
              <l-icon
                :icon-url="icon.type.gold"
                :shadow-url="icon.shadowUrl"
                :icon-size="icon.iconSize"
                :icon-anchor="icon.iconAnchor"
                :popup-anchor="icon.popupAnchor"
                :shadow-size="icon.shadowSize"
              />
              <l-popup> 你的位置 </l-popup>
            </l-marker>
            <!-- 創建標記點 -->
            <l-marker
              :lat-lng="item.local"
              v-for="item in selected_data"
              :key="item.id"
            >
              <!-- 標記點樣式判斷 -->
              <l-icon
                :icon-url="icon.type.black"
                :shadow-url="icon.shadowUrl"
                :icon-size="icon.iconSize"
                :icon-anchor="icon.iconAnchor"
                :popup-anchor="icon.popupAnchor"
                :shadow-size="icon.shadowSize"
              />
              <!-- 彈出視窗 -->
              <l-popup>
                <p style="font-size: 20px; font-weight: bold">
                  {{ item.name }}
                </p>

                <span style="font-size: 16px; font-weight: bold; color: red">
                  剩餘數量 大人 {{ item.mask_adult }} / 兒童
                  {{ item.mask_child }}
                </span>
                <br />
                <span
                  >地址 {{ item.address }}<br />
                  電話 {{ item.phone }}<br />
                  最後更新時間 {{ item.updated }}
                </span>
              </l-popup>
            </l-marker>
          </l-map>
        </div>
      </v-col>
    </v-row>
  </v-container>
</template>

<script>
import CountyTownData from "../assets/CountyTownData.json";
import axios from "axios";
import { LMap, LTileLayer, LMarker, LPopup, LIcon } from "vue2-leaflet";
import "leaflet/dist/leaflet.css";

export default {
  name: "MaskMap",
  components: {
    LMap,
    LTileLayer,
    LMarker,
    LPopup,
    LIcon,
  },
  data: () => ({
    // county town data
    county_town_data: [],
    town_list: [],

    selected_county: {},
    selected_town: {},

    // mask data
    mask_data: [],

    // map
    selected_data: [],
    zoom: 14,
    center: [25.042474, 121.513729],
    your_location: [25.042474, 121.513729],

    url: "https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png",
    attribution: `© <a href="http://osm.org/copyright">OpenStreetMap</a> contributors`,
    options: {
      // zoomControl: false,
      scrollWheelZoom: false,
    },
    icon: {
      type: {
        black:
          "https://cdn.rawgit.com/pointhi/leaflet-color-markers/master/img/marker-icon-2x-black.png",
        gold: "https://cdn.rawgit.com/pointhi/leaflet-color-markers/master/img/marker-icon-2x-gold.png",
      },
      shadowUrl:
        "https://cdnjs.cloudflare.com/ajax/libs/leaflet/0.7.7/images/marker-shadow.png",
      iconSize: [25, 41],
      iconAnchor: [12, 41],
      popupAnchor: [1, -34],
      shadowSize: [41, 41],
    },
  }),
  methods: {
    async getAllMaskData() {
      try {
        let url =
          "https://raw.githubusercontent.com/kiang/pharmacies/master/json/points.json";
        let result = await axios.get(url);
        if (result.status === 200) {
          this.mask_data = result.data.features;
        }
      } catch (error) {
        console.log(error);
      }
    },

    showTown() {
      this.town_list = this.selected_county.AreaList;
      this.selected_town = {};
      this.updateMaskData();
    },

    updateMaskData() {
      let result = [];
      let vm = this;
      if (this.mask_data.length > 0) {
        let data = this.mask_data.filter((mask) => {
          if (Object.keys(vm.selected_town).length === 0) {
            // return county
            return mask.properties.county === vm.selected_county.CityName;
          } else {
            // return town
            return (
              mask.properties.county === vm.selected_county.CityName &&
              mask.properties.town === vm.selected_town.AreaName
            );
          }
        });
        for (let i = 0; i < data.length; i++) {
          let result_pre = data[i].properties;
          result_pre.local = [
            data[i].geometry.coordinates[1],
            data[i].geometry.coordinates[0],
          ];
          result.push(result_pre);
        }
      }
      console.log(result);
      this.selected_data = result;
      if (this.selected_data.length > 0) {
        this.center = this.selected_data[0].local;
      }
    },
  },
  async created() {
    // load mask data
    await this.getAllMaskData();

    // load county-town data
    this.county_town_data = CountyTownData;
    if (this.county_town_data.length > 0) {
      this.selected_county = this.county_town_data[0];
      this.town_list = this.selected_county.AreaList;
      this.selected_town = this.town_list[0];
      this.updateMaskData();
    }
  },
  mounted() {
    // 等地圖創建後執行
    this.$nextTick(() => {
      // 獲得目前位置
      navigator.geolocation.getCurrentPosition((position) => {
        const p = position.coords;
        // 將中心點設為目前的位置
        this.your_location = [p.latitude, p.longitude];
        // 將目前的位置的標記點彈跳視窗打開
        this.$refs.location.mapObject.openPopup();
      });
    });
  },
  computed: {},
};
</script>

<style>
.vue2leaflet-map {
  z-index: 1;
}
</style>
