<template>
  <div>
    <div id="map">
      <gmap-map
        :center="center"
        :zoom="12"
        :options="mapOptions"
        :style="mapStyle"
        ref="mapRef"
      >
        <GmapMarker
          :key="index"
          v-for="(m, index) in markers"
          :position="m.position"
          :clickable="false"
          :draggable="false"
          :icon="m.icon"
        />
      </gmap-map>
    </div>
    <div class="container mb-15">
      <v-card v-if="noorder" elevation="2" class="mx-1"
        ><v-card-text>目前沒有接到訂單喔~</v-card-text></v-card
      >
      <v-card elevation="2" class="mx-1" v-else>
        <v-card-text>
          <div class="font-weight-bold ml-8 mb-2">訂單狀態</div>

          <v-timeline align-top dense>
            <v-timeline-item
              v-for="(status, index) in orderstatus"
              :key="index"
              :color="status.color"
              small
            >
              <div class="font-weight-normal">
                <strong>{{ status.status }}</strong>
              </div>
            </v-timeline-item>
          </v-timeline>
        </v-card-text>
        <v-card-text> 餐廳: {{ orderinfo.restaurant.name }} </v-card-text>
        <v-card-text> 送達地址: {{ orderinfo.customer_address }} </v-card-text
        ><v-card-text
          ><v-btn icon :href="mapurl">
            <v-icon>fas fa-map-marked-alt</v-icon>
          </v-btn></v-card-text
        >
        <v-card-text> 餐點內容: </v-card-text>
        <div v-for="dish in orderinfo.items" :key="dish.id" class="ml-7">
          {{ dish.name }} x {{ dish.amount }}
        </div>
        <v-card-text> 餐點費用: {{ orderinfo.food_price }} </v-card-text>
        <v-card-text> 外送費用: {{ orderinfo.delivery_fee }} </v-card-text>
        <v-card-text>
          總共: {{ orderinfo.food_price + orderinfo.delivery_fee }}
        </v-card-text>
        <v-card-text> 預估剩餘時間: {{ remaintime }} 分鐘</v-card-text>
        <v-card-text>
          外送員電話: {{ orderinfo.deliveryMan.phone }}
        </v-card-text>
        <v-card-text> 備註: {{ orderinfo.note }} </v-card-text>
      </v-card>
      <v-speed-dial
        v-show="speedshow"
        v-model="fab"
        :top="false"
        :bottom="true"
        :right="true"
        :left="false"
        :direction="'top'"
        :open-on-hover="false"
        :transition="'slide-y-reverse-transition'"
        :fixed="true"
        class="mb-15 mr-3"
      >
        <template v-slot:activator>
          <v-btn v-model="fab" color="primary" dark fab>
            <v-icon v-if="fab">mdi-close</v-icon>
            <v-icon v-else>fas fa-chevron-up</v-icon>
          </v-btn>
        </template>
        <v-btn fab dark small color="green" @click="changeorderstatus">
          <v-icon>fas fa-check</v-icon>
          <div class="fab-text-custom green">{{ changestatusmsg }}</div>
        </v-btn>
      </v-speed-dial>
    </div>
  </div>
</template>

<script src="https://js.pusher.com/7.0/pusher.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/vue/dist/vue.js"></script>
<script>
import axios from "axios";
import Pusher from "pusher-js";
import {
  deliveryManGetOrderstatusAPI,
  deliveryManSendLocationAPI,
  deliveryManAddressToLocationAPI,
  deliveryManGetDeliveryTimeIDAPI,
  deliveryManSwitchOrderStateAPI,
} from "../../api";

export default {
  props: ["id"],
  data: () => ({
    mapurl: "",
    speedshow: true,
    fab: false,
    changestatusmsg: "",
    config: {},
    mapStyle:
      "width: " +
      window.innerWidth +
      "px; height: " +
      window.innerHeight +
      "px;",
    mapOptions: { disableDefaultUI: true, clickableIcons: false },
    markers: [
      { position: { lat: 25, lng: 121 }, icon: require("../house.png") },
      { position: { lat: 25, lng: 121 }, icon: require("../scooter.png") },
    ],
    center: { lat: 45.508, lng: -73.587 },
    orderinfo: {},
    remaintime: "loading",
    timearray: [],
    noorder: true,
    orderstatus: [
      {
        status: "已送達/等待顧客評價",
        color: "grey",
      },
      {
        status: "送餐中",
        color: "grey",
      },
      {
        status: "餐點製作中",
        color: "grey",
      },
    ],
  }),
  created() {},
  mounted() {
    window.setInterval(() => {
      this.update();
    }, 10000);
    this.config = {
      headers: {
        Authorization: "Bearer " + this.$store.getters.getAccessToken,
      },
    };
    this.$emit("changefocus", "order");
    this.onResize();
    this.updatelocate();
    this.updateorder();
    this.updatebound();
    this.$emit("showSnackBar", "獲取位置中...請稍後");
    console.log(this.markers);
    console.log(this.markers.length);
    console.log(this.markers[2]);
  },
  methods: {
    changeorderstatus() {
      console.log("changeorderstatus" + this.changestatusmsg);
      deliveryManSwitchOrderStateAPI(this.config, {
        status: this.orderinfo.status + 1,
        id: this.orderinfo.id,
      })
        .then((resp1) => {
          if (resp1.data.data.status === 3) this.$router.go(0);
          this.updateorder();
        })
        .catch((err) => {
          console.log(err);
          if (err.response.status === 401) {
            this.$emit("showSnackBar", "401error");
            console.log(err);
          } else if (err.response.status === 404) {
            this.$emit("showSnackBar", "404error");
            console.log(err);
          }
          console.log(err);
        });
    },
    update() {
      this.updatelocate();
      this.updateorder();
      this.updatebound();
    },
    updatebound() {
      this.$refs.mapRef.$mapPromise.then((map) => {
        var bounds = new google.maps.LatLngBounds();
        for (let i = 0; i < this.markers.length; i++) {
          bounds.extend(this.markers[i].position);
          console.log("in");
        }

        //now fit the map to the newly inclusive bounds
        if (this.markers.length > 0) map.fitBounds(bounds);
      });
    },
    onResize() {
      this.mapStyle =
        "width:" +
        window.innerWidth +
        "px;  height: " +
        window.innerHeight / 4 +
        "px;";
    },
    updatelocate() {
      navigator.geolocation.getCurrentPosition((position) => {
        this.center = {
          lat: position.coords.latitude,
          lng: position.coords.longitude,
        };
        deliveryManSendLocationAPI(
          {
            latitude: position.coords.latitude,
            longitude: position.coords.longitude,
          },
          this.config
        )
          .then((resp) => {
            console.log(resp.status);
          })
          .catch((error) => {
            console.log(error);
          });
      });
    },
    updateorder() {
      deliveryManGetOrderstatusAPI(this.config)
        .then((resp) => {
          if (resp.status === 200) {
            this.noorder = false;
            console.log(resp.data.data.customer.latitude);
            // update map markers
            this.orderinfo = resp.data.data;
            if (this.orderinfo.status >= 1) {
              this.orderstatus[2].color = "deep-purple";
              this.changestatusmsg = "通知顧客已取餐";
            }
            if (this.orderinfo.status >= 2) {
              this.orderstatus[1].color = "deep-purple";
              this.changestatusmsg = "通知顧客已送達";
            }
            if (this.orderinfo.status >= 3) {
              this.orderstatus[0].color = "deep-purple";
              this.speedshow = false;
            }
            let config = {
              params: { address: this.orderinfo.customer_address },
              headers: {
                Authorization: "Bearer " + this.$store.getters.getAccessToken,
              },
            };
            deliveryManAddressToLocationAPI(config)
              .then((resp1) => {
                console.log(resp1.data.results[0].geometry.location.lat);
                const customerMarker = {
                  lat: resp1.data.results[0].geometry.location.lat,
                  lng: resp1.data.results[0].geometry.location.lng,
                };
                this.mapurl =
                  "https://www.google.com/maps/search/?api=1&query=" +
                  resp1.data.results[0].geometry.location.lat +
                  "," +
                  resp1.data.results[0].geometry.location.lng;
                const mapMarkerIcon = require("../house.png");
                this.markers[0] = {
                  position: customerMarker,
                  icon: mapMarkerIcon,
                };
              })
              .catch((err) => {
                console.log(err);
                if (err.response.status === 401) {
                  this.$emit("showSnackBar", "401error");
                  console.log(err);
                } else if (err.response.status === 404) {
                  this.$emit("showSnackBar", "404error");
                  console.log(err);
                }
                console.log(err);
              });
            navigator.geolocation.getCurrentPosition((position) => {
              const dliverManMarker = {
                lat: position.coords.latitude,
                lng: position.coords.longitude,
              };
              const mapMarkerIcon = require("../scooter.png");
              this.markers[1] = {
                position: dliverManMarker,
                icon: mapMarkerIcon,
              };
              //remain time
              deliveryManGetDeliveryTimeIDAPI({
                headers: {
                  Authorization: "Bearer " + this.$store.getters.getAccessToken,
                },
                params: {
                  ori_address:
                    resp.data.data.deliveryMan.latitude +
                    "," +
                    resp.data.data.deliveryMan.longitude,
                  des_address: this.orderinfo.customer_address,
                },
              })
                .then((res) => {
                  console.log(res.data);
                  // add food making time
                  this.orderinfo.items.forEach((element) => {
                    this.timearray.push(element.making_time);
                  });

                  if (this.orderinfo.status === 1) {
                    this.remaintime =
                      parseInt(
                        res.data.data.rows[0].elements[0].duration.text
                      ) + Math.max(...this.timearray);
                  }
                  if (this.orderinfo.status === 2) {
                    this.remaintime = parseInt(
                      res.data.data.rows[0].elements[0].duration.text
                    );
                  }
                  console.log("done get time");
                })
                .catch((error) => {
                  console.log(error);
                });
            });
          }
        })
        .catch((err) => {
          console.log(err);
          if (err.response.status === 401) {
            this.$emit("showSnackBar", "401error");
            console.log(err);
          } else if (err.response.status === 404) {
            console.log(err);
          }
          console.log(err);
        });
    },
  },
};
</script>

<style scoped>
.fab-text-custom {
  position: absolute;
  right: 50px;
  background-color: rgba(0, 0, 0, 0.5);
  padding: 10px;
  box-shadow: 0px 3px 5px -1px rgba(0, 0, 0, 0.2),
    0px 6px 10px 0px rgba(0, 0, 0, 0.14), 0px 1px 18px 0px rgba(0, 0, 0, 0.12);
  border-radius: 2px;
}
</style>