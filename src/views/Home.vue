<template>
  <div class="container">
    <div class="navbar">
      <div class="title">
        <i class="material-icons">insert_drive_file</i>
        <span> Report</span>
        <span class="sub-title" v-if="!isLocality">
          Individual pin code report for {{ inputValue }}
        </span>
        <span class="sub-title" v-else>Locality ID {{ inputValue }}</span>
      </div>
      <div class="navBtn">
        <input
          class="inputBox"
          v-if="isLocality"
          type="text"
          v-model="inputValue"
          placeholder="enter locality Id"
        />
        <input
          class="inputBox"
          v-else
          type="text"
          v-model="inputValue"
          placeholder="enter pincode"
        />
        <button @click="changeLocality" class="btn">
          {{ isLocality == false ? "Pincode" : "Locality" }}
        </button>
      </div>
    </div>
    <div class="flex-container">
      <div class="flex-item stats">
        <div class="row">
          <div class="col">
            <span class="count">{{
              fetchStats.population === undefined
                ? "Not Available"
                : fetchStats.population
            }}</span>
            Population
          </div>
          <div class="col">
            <span class="count">{{
              fetchStats.households === undefined
                ? "Not Available"
                : fetchStats.households
            }}</span>
            HouseHolds
          </div>
        </div>
        <div v-if="isLocality" class="row address">
          <div class="col">
            Locality:
            <b>{{
              fetchStats.locality === undefined
                ? "Not Available"
                : fetchStats.locality
            }}</b>
          </div>
          <div class="col">
            City:
            <b>{{
              fetchStats.city === undefined ? "Not Available" : fetchStats.city
            }}</b>
          </div>
        </div>
        <div v-else class="row address">
          <div class="col">
            District:
            <b>{{
              fetchStats.district_n === undefined
                ? "Not Available"
                : fetchStats.district_n
            }}</b>
          </div>
          <div class="col">
            Country:
            <b>India</b>
          </div>
        </div>
      </div>
      <div class="flex-item">
        <!-- {{ geoGjsonData }} -->
        <l-map :zoom="zoom" :center="center" ref="myMap">
          <l-tile-layer :url="url"></l-tile-layer>
          <l-geo-json
            v-if="geoGjsonData !== undefined && geoGjsonData.type !== undefined"
            :geojson="geoGjsonData"
          >
          </l-geo-json>
        </l-map>
      </div>
    </div>
    <div class="flex-container">
      <div class="charts">
        <span class="expenditure-label">Demographics</span>
        <doughnut-chart
          v-if="
            fetchStats !== undefined &&
              fetchStats.datacollection !== undefined &&
              fetchStats.datacollection.datasets.length !== 0
          "
          :chart-data="fetchStats.datacollection"
        ></doughnut-chart>
        <!-- {{ fetchStats.datacollection }} -->
        <div
          class="row"
          v-if="
            (fetchStats !== undefined &&
              fetchStats.datacollection == undefined) ||
              (fetchStats.datacollection !== undefined &&
                fetchStats.datacollection.datasets.length === 0)
          "
        >
          <div class="col no-stats-label">
            <span class="count">No Data Available </span>
            <span class="label">Monthly Income Distribution</span>
          </div>
        </div>
      </div>
      <div class="charts">
        <div
          v-if="isLocality || fetchStats.barDatacollection === undefined"
          class="row"
        >
          <div class="col no-stats-label">
            <span class="count">No Data Available</span>
            <span class="label">Expenditure for {{ inputValue }}</span>
          </div>
        </div>
        <bar-chart
          v-if="!isLocality && fetchStats.barDatacollection !== undefined"
          :chart-data="fetchStats.barDatacollection"
        ></bar-chart>
      </div>
    </div>
  </div>
</template>

<script>
import constants from "@/constants/constants.js";
import DoughnutChart from "@/charts/DoughnutChart.js";
import BarChart from "@/charts/BarChart.js";
import { LMap, LTileLayer, LGeoJson } from "vue2-leaflet";
export default {
  name: "home",
  components: {
    DoughnutChart,
    BarChart,
    LMap,
    LTileLayer,
    LGeoJson
  },
  data() {
    return {
      isLocality: true,
      isCDNDataLoaded: false,
      inputValue: "17",
      pinCodeData: {},
      localityData: {},
      incomeData: [],
      expenditureData: [],
      url: "http://{s}.tile.osm.org/{z}/{x}/{y}.png",
      zoom: 8,
      center: [12.9716, 77.5946],
      geoGjsonData: {}
    };
  },
  computed: {
    fetchStats() {
      let stats = {};
      if (
        this.inputValue !== "" &&
        this.isLocality &&
        this.localityData.features !== undefined
      ) {
        this.localityData.features.forEach(feature => {
          if (feature.attributes.locality_i === parseInt(this.inputValue, 10)) {
            stats = feature.attributes;
            stats.datacollection = { labels: [], datasets: [] };
            stats.datacollection.labels.push(stats.locality);
            let datasetObj = { data: [] };
            datasetObj.label = stats.locality;
            datasetObj.backgroundColor = "cadetblue";

            // fetch monthly income of locality
            let foundObj = this.incomeData.find(
              income => income.locality === stats.locality
            );
            if (foundObj == undefined) {
              return stats;
            }
            datasetObj.data.push(foundObj.income);
            stats.datacollection.datasets.push(datasetObj);
          }
        });
        return stats;
      }
      if (
        this.inputValue !== "" &&
        !this.isLocality &&
        this.pinCodeData.features !== undefined
      ) {
        this.pinCodeData.features.forEach(feature => {
          if (feature.attributes.pincode === parseInt(this.inputValue, 10)) {
            stats = feature.attributes;
          }
        });
        let foundExpenditure = this.expenditureData.find(expenditure => {
          if (expenditure.pincode === parseInt(this.inputValue, 10)) {
            return expenditure;
          }
        });
        if (foundExpenditure === undefined) {
          return stats;
        }
        // console.log("foundExpenditure", foundExpenditure);
        stats.barDatacollection = { labels: [], datasets: [] };
        let datasetObj = { data: [] };
        datasetObj.label = "Expenditure for " + this.inputValue;
        datasetObj.backgroundColor = "cadetblue";

        for (let [key, value] of Object.entries(foundExpenditure)) {
          if (key !== "pincode") {
            stats.barDatacollection.labels.push(key);
            datasetObj.data.push(value);
          }
        }
        stats.barDatacollection.datasets.push(datasetObj);
        // console.log("stats.barDatacollection", stats.barDatacollection);
        return stats;
      }
      return {};
    }
  },
  methods: {
    changeLocality() {
      this.inputValue = "";
      this.isLocality = !this.isLocality;
    },
    prepareGeoJsonData() {
      let geoGjsonData = { features: [] };
      geoGjsonData.type = "FeatureCollection";
      // /iterate localityData and add features
      this.localityData.features.forEach(feature => {
        let featureObj = { geometry: { coordinates: [] } };
        featureObj.type = "Feature";
        featureObj.geometry.type = "MultiPolygon";
        featureObj.geometry.coordinates = feature.geometry.rings;
        geoGjsonData.features.push(featureObj);
      });
      this.geoGjsonData = geoGjsonData;
      // LMap.geoJSON(this.geoGjsonDatas).addTo(map);
      // console.log("prepared geoGjsonData", JSON.stringify(this.geoGjsonData));
    }
  },
  mounted() {
    this.$nextTick(() => {
      this.axios.get(constants.PINCODE_CDN_URL).then(res => {
        // console.log("called1");
        this.pinCodeData = res.data;
        // console.log("this.pinCodeData", this.pinCodeData);
      });
      this.axios.get(constants.LOCALITY_CDN_URL).then(res => {
        this.localityData = res.data;
        // console.log("this.localityData", this.localityData);
        // this.prepareGeoJsonData();
      });
      this.axios.get(constants.INCOME_CDN_URL).then(res => {
        this.incomeData = res.data;
        // console.log("this.incomeData", this.incomeData);
      });
      this.axios.get(constants.EXPENDITURE_CDN_URL).then(res => {
        this.expenditureData = res.data;
        // console.log("this.expenditureData", this.expenditureData);
      });
    });
  }
};
</script>
<style scoped>
.flex-container {
  display: flex;
  flex-direction: row;
  /* flex-wrap: wrap; */
}
.flex-item {
  flex: 50%;
  height: 35vh;
  /* border: solid 1px gray; */
}
.charts {
  flex: 50%;
  /* border: solid 1px gray; */
  height: 50vh;
}
.navbar {
  height: 10vh;
  padding-top: 10px;
  background-color: cadetblue;
}
.title {
  display: block;
  float: left;
  position: absolute;
  padding-left: 15px;
  /* padding: 10px; */
  color: white;
  font-size: 25px;
}
.container {
  margin: 0px;
  padding: 0px;
}
.stats {
  background-color: #e6f0ff;
}
.btn {
  padding: 5px;
  color: #fff;
  font-weight: normal;
  background-color: #000;
}
.navBtn {
  float: right;
  padding-right: 10px;
}
.inputBox {
  height: 20px;
  margin: 2px;
  padding: 3px;
}
.row {
  padding: 5px;
  display: flex;
  justify-content: center;
}
.col {
  flex: 50%;
  /* border: solid 1px black; */
}
.count {
  display: block;
  font-size: 20px;
  color: cadetblue;
  font-weight: bold;
}
.address {
  background-color: #fff;
  color: cadetblue;
  padding: 10px;
}
.expenditure-label {
  color: cadetblue;
  float: left;
  font-size: 30px;
}
.no-stats-label {
  padding-top: 20%;
}
.label {
  color: cadetblue;
}
.sub-title {
  display: block;
  font-size: small;
  /* clear: both; */
}
@media screen and (max-width: 576px) {
  .flex-container,
  .navbar {
    display: block;
  }
}
</style>
