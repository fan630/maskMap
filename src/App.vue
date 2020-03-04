<template>
    <div id="app">
        <div class="bg-secondary py-1 d-flex justify-content-between px-2">
          <div class="h3 mb-0 text-white" style="line-height:68px">
              口罩地圖
          </div>
          <div style="line-height:68px">
            <b-button v-b-modal.modal-lg class="mr-1" style="border-color:#fff; border-width:2px">
                <div>
                  <i class="far fa-question-circle fa-1x"></i>
                </div>
            </b-button>
            <b-modal id="modal-lg" size="md" title="購買規定 (2020/02/18)" hide-footer>
                 <div class="text-center">
                                    <img src="https://imgur.dcard.tw/CPYUzS1.jpg" alt="" srcset="" width="70%" height="50%">
                 <p class="text-info">
                    <a href="https://www.cdc.gov.tw/Bulletin/Detail/wycLegOplsUwGsH6_SPAXg?typeid=9" target="_blank">
                        資料來源:衛生福利部疾病管制署
                    </a>
                 </p>
                 <p class="text-success my-1">藥局顯示為淺綠色, 表示還有足量庫存</p>
                 <div>
                    <span>
                      <img src="https://raw.githubusercontent.com/pointhi/leaflet-color-markers/master/img/marker-icon-green.png" alt="" srcset="">
                      貨源充足
                    </span>
                    <span>
                      <img src="https://raw.githubusercontent.com/pointhi/leaflet-color-markers/master/img/marker-icon-red.png" alt="" srcset="">
                      貨源不足
                    </span>
                    <span>
                      <img src="https://raw.githubusercontent.com/pointhi/leaflet-color-markers/master/img/marker-icon-grey.png" alt="" srcset="">
                      已經沒貨
                    </span>
                 </div>
                 </div>

            </b-modal>
          </div>
        </div>
        <div class="row no-gutters">
              <div class="col-md-3 toolbox">
                    <div class="sticky-top shadow-sm p-2">
                      <div class="citySelect">
                        <select class="custom-select custom-select-md mb-3" v-model="select.filter">
                          <option selected>Open this select menu</option>
                          <option :value="item" v-for="(item, key) in cityFilter" :key="key">{{item}}</option>
                        </select>
                      </div>
                      <div class="TownSelect">
                        <select class="custom-select custom-select-md mb-3" 
                                v-model="select.filter2"
                          >
                          <option :value="item" 
                                  v-for="(item, key) in townFilter" 
                                  :key="key">
                                  {{item}}
                          </option>
                        </select>
                      </div>

                      <div class="input-group mb-3">
                        <input type="text" class="form-control" 
                            placeholder="請輸入欲查詢的縣市和區域" aria-describedby="button-addon2"
                            v-model="inputAddress">
                        <div class="input-group-append">
                          <button class="btn btn-outline-secondary btn-sm" 
                                  type="button" id="button-addon2" 
                                  @click="filterAddress"
                                  >
                              <div>
                                <i class="fas fa-search"></i>
                              </div>
                            </button>
                        </div>
                      </div>

                      <div class="mr-1 text-sm-center text-muted">總共找到<span>{{filterData.length}}</span>家相符的藥局</div>
                    </div>
                    <div class="cardGroup">
                      <Card
                          :store="item"
                          v-for="(item, key) in filterData"
                          :key="key"
                          @switchcard="switchGeo"
                          @switchshare="switchNote"
                        >
                        </Card>
                    </div>
              </div>
              <loading :active.sync="isLoading">
                  <div class="col-md-9">
                      <i class="fas fa-sync fa-spin fa-3x" v-if="status.fileUploading">
                        </i>
                  </div>
              </loading>
              <b-modal ref="my-modal" hide-footer title="備註" 
                  header-bg-variant="info" header-text-variant="white"
                >
                <div class="d-block text-left">
                  <p>{{ noteMessage }}</p>
                </div>
              </b-modal>
              <div class="col-md-9">
                  <div id="map">
                  </div>
              </div>
            </div>
    </div>
</template>

<script>
import Card from "./components/Card.vue";
import L from "leaflet";
import "leaflet/dist/leaflet.css";
import "leaflet.markercluster";

let osmMap = {};

let iconsConfig = {
  iconSize: [25, 41],
  iconAnchor: [12, 41],
  popupAnchor: [1, -34],
  shadowSize: [41, 41]
};

let iconData = {
  green: new L.Icon({
    iconUrl:
      "https://cdn.rawgit.com/pointhi/leaflet-color-markers/master/img/marker-icon-2x-green.png",
    shadowUrl:
       'https://cdnjs.cloudflare.com/ajax/libs/leaflet/0.7.7/images/marker-shadow.png',
    ...iconsConfig
  }),
  grey: new L.Icon({
    iconUrl:
      "https://cdn.rawgit.com/pointhi/leaflet-color-markers/master/img/marker-icon-2x-grey.png",
    shadowUrl:
       'https://cdnjs.cloudflare.com/ajax/libs/leaflet/0.7.7/images/marker-shadow.png',
    ...iconsConfig
  }),
  red: new L.Icon({
    iconUrl:
      "https://cdn.rawgit.com/pointhi/leaflet-color-markers/master/img/marker-icon-2x-red.png",
    shadowUrl:
       'https://cdnjs.cloudflare.com/ajax/libs/leaflet/0.7.7/images/marker-shadow.png',
    ...iconsConfig
  })
};

export default {
  name: "App",
  data() {
    return {
      dataAry: [],
      city: [],
      cityFilter: [],
      newAry:[], 
      town:[],
      townFilter:[],
      geoInfo: [],
      markers: [],
      inputAddress:'', 
      noteMessage:'',
      isLoading: true,
      select:{
          filter: "新北市",
          filter2:"新店區"
      },
      status: {
        fileUploading: true
      },
      iconData
    };
  },
  methods: {
    initialMap() {
        //navigator.geolocation.getCurrentPosition((position) => {
        //console.log(position.coords.latitude, position.coords.longitude);  
        osmMap = L.map("map", {
          //center: [position.coords.latitude, position.coords.longitude], // localhost ok
          center: [24.956111, 121.536500],// network定位在碧潭
          zoom: 18
        });

        L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png", {
          attribution:'Map data ©<a href="http://www.openstreetmap.org/copyright">OpenStreetMap</a>|Created by <a href="https://github.com/fan630?tab=repositories" target="_blank">fan630</a>',
          maxZoom: 19
        }).addTo(osmMap)
      //})
    },
    getUser(){
      if ("geolocation" in navigator) {
        /* geolocation is available */
        navigator.geolocation.getCurrentPosition(displayLocationInfo);
        
        function displayLocationInfo(position) {
          const lng = position.coords.longitude;
          const lat = position.coords.latitude;

          console.log(`longitude: ${ lng } | latitude: ${ lat }`);
        }
      } else {
        /* geolocation IS NOT available  */
        console.log('geolocation IS NOT available on your browser');
      }

    }, 

    switchGeo(store) {
      let newLat = store.geometry.coordinates[1];
      let newLng = store.geometry.coordinates[0];
      osmMap.flyTo(new L.LatLng(newLat, newLng));

       L.marker([newLat, newLng]).addTo(osmMap)
         .bindPopup(
           `<h4>${store.properties.name}</h4>
               <p>
                  成人口罩:<mark>${store.properties.mask_adult}</mark><br>
                  兒童口罩:${store.properties.mask_child}<br>
                  診所電話:
                  <a href="tel:+886-${store.properties.phone}">${store.properties.phone}</a>
               </p>`
         ).openPopup();
    },

    switchNote(store){
        const vm = this;
        vm.noteMessage = store.properties.note;
        vm.$refs['my-modal'].toggle('#toggle-btn')
    },

    filterColor(count) {
      switch (true) {
        case count >= 10:
          return this.iconData.green;
          break;
        case count < 10 && count > 0:
          return this.iconData.red;
          break;
        case count == 0:
          return this.iconData.grey;
          break;
      }
    },

    dataCreated() {
      this.status.fileUploading = true
      this.isLoading = true

      let markers = new L.MarkerClusterGroup().addTo(osmMap);
      for (let i = 0; i < this.geoInfo.length; i++) {
        this.status.fileUploading = false
        this.isLoading = false  
        markers.addLayer(
          L.marker([this.geoInfo[i].lng, this.geoInfo[i].lat], {
            icon: this.filterColor(this.geoInfo[i].adult)
          }).bindPopup(`<h3>${this.geoInfo[i].name}</h3>
                              <p>
                                成人口罩:${this.geoInfo[i].adult}  
                                兒童口罩:${this.geoInfo[i].child}
                                診所電話:${this.geoInfo[i].phone}
                              </p>`)
        );
      }
      osmMap.addLayer(markers);
    }, 

    filterAddress(){
        this.newAry.length = 0 
        return this.dataAry.filter(item => {
            if(item.properties.address.includes(this.inputAddress)){
                this.select.filter = ''
                this.select.filter2 = ''
                this.newAry.push(item)
            }
        })
    }
  }, 
  computed: {
    filterData() {
      let vm = this;
      if (vm.select.filter === "Open this select menu") {
        return vm.dataAry;
      } else if(this.select.filter === '' && this.select.filter2 === ''){
        return vm.newAry; 
      }else{
        vm.town.length = 0
        vm.dataAry.find(item => {
            if (vm.select.filter === item.properties.county){
                vm.town.push(item.properties.town)
            }
        })
        vm.townFilter = vm.town.filter((item, index, arr) => {
          return vm.town.indexOf(item) === index;
        });
        return vm.dataAry.filter(item => {
          return item.properties.county === vm.select.filter && item.properties.town == vm.select.filter2
        });
      }
    },
  },
  mounted() {
      this.initialMap()
      // this.getUser()
      this.axios.get('https://raw.githubusercontent.com/kiang/pharmacies/master/json/points.json?fbclid=IwAR0TSsgLIAH10XrE6fFk7bRmXyxXd_FK4NZO39yOkcS28rpVfCkQZJit0Dc')
      .then((res) => {
        this.dataAry = res.data.features;

        this.dataAry.forEach(item => {
          return this.city.push(item.properties.county);
        });
        
        this.cityFilter = this.city.filter((item, index, arr) => {
          return this.city.indexOf(item) === index;
        });

        let index = this.cityFilter.findIndex(item => {
          return item === ''
        })

        this.cityFilter.splice(index, 1);

        this.dataAry.forEach(item => {
          return this.geoInfo.push({
            name: item.properties.name,
            lat: item.geometry.coordinates[0],
            lng: item.geometry.coordinates[1],
            adult: item.properties.mask_adult,
            child: item.properties.mask_child, 
            phone: item.properties.phone
          });
        });

        this.dataCreated();

    }) 
  },
  components: {
    Card, 
  }
};
</script>

<style lang="scss">
@import "~bootstrap/scss/bootstrap.scss";
@import '~bootstrap-vue/src/index.scss';

// .toggleBtn{
//   position: absolute;
//   z-index: 1;
//   left: 22%;
//   top: 40%;
// }




// @media (max-width: 767.98px) {
//     .row > .toolbox{

//     }
// }

body::-webkit-scrollbar {
    display: none;
}

p{
  margin-bottom: 0px;
}


#map {
  height: 100vh;
  overflow: hidden;
}

.marker-cluster-small {
  background-color: rgba(181, 226, 140, 0.6);
}

// .marker-cluster-small::after {
//     content: " ";
//     background: url('https://www.mapsmarker.com/wp-content/uploads/maps-marker-pro/icons/hospital-building.png');
//     height: 36px;
//     width: 36px;
//     background-repeat: no-repeat;
//     display: inline-block;
// }

.marker-cluster-small div {
  background-color: rgba(110, 204, 57, 0.6);
}

.marker-cluster-medium {
  background-color: rgba(241, 211, 87, 0.6);
}
.marker-cluster-medium div {
  background-color: rgba(240, 194, 12, 0.6);
}

.marker-cluster-large {
  background-color: rgba(253, 156, 115, 0.6);
}
.marker-cluster-large div {
  background-color: rgba(241, 128, 23, 0.6);
}

.marker-cluster {
  background-clip: padding-box;
  border-radius: 20px;
}
.marker-cluster div {
  width: 30px;
  height: 30px;
  margin-left: 5px;
  margin-top: 5px;
  text-align: center;
  border-radius: 15px;
  font: 12px "Helvetica Neue", Arial, Helvetica, sans-serif;
}
.marker-cluster span {
  line-height: 30px;
}

.toolbox{
  height:100vh; 
  overflow-y: auto;
}

.toolbox > p{
  background-color: #ACBDBC;
}

.sticky-top{
  background-color: #E0E2E0;
}

.text-muted > span {
  font-size: 20px;
  color: #3986f7;
  font-weight: 700;
  margin: 0 5px;
}



</style>
