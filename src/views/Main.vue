<template>
  <div id="app">
    <!-- <Loader v-if="isLoader" /> -->

    <div class="sensors-screen">
      <div class="sensors-screen-layers">
        <div class="sensors-screen-layers--center">
          <Header :localeCurrent="$i18n.locale" :city="city" />

          <div class="container sensors-container">
            <Measures :current="type.toLowerCase()" />
            <ColorfulScale />

            <template v-if="point">
              <MessagePopup
                v-if="point.data.message"
                @close="handlerClose"
                :data="point.data"
              />
              <SensorPopup
                v-else
                :sender="point.sender"
                :sensor_id="point.sensor_id"
                :log="point.log"
                :model="point.model"
                :address="point.address"
                :geo="point.geo"
                :type="type.toLowerCase()"
                @modal="handlerModal"
                @close="handlerClose"
              />
            </template>

            <Map
              :type="type"
              :zoom="zoom"
              :lat="lat"
              :lng="lng"
              :availableWind="provider === 'ipfs'"
              @clickMarker="handlerClick"
              @city="handlerChangeCity"
            />
          </div>

          <Footer
            :currentProvider="provider"
            :canHistory="canHistory"
            @history="handlerHistory"
          />
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import ColorfulScale from "../components/colorfulScale/ColorfulScale.vue";
import Footer from "../components/footer/Footer.vue";
import Header from "../components/header/Header.vue";
import Loader from "../components/Loader.vue";
import Map from "../components/Map.vue";
import Measures from "../components/measures/Measures.vue";
import MessagePopup from "../components/message/MessagePopup.vue";
import SensorPopup from "../components/sensor/SensorPopup.vue";
import config from "../config";
import * as providers from "../providers";
import { instanceMap } from "../utils/map/instance";
import * as markers from "../utils/map/marker";
import { getAddressByPos } from "../utils/map/utils";
import { getMapPosiotion } from "../utils/utils";

const mapPosition = getMapPosiotion();

export default {
  props: {
    provider: {
      default: "remote",
    },
    type: {
      default: "pm10",
    },
    zoom: {
      default: mapPosition.zoom,
    },
    lat: {
      default: mapPosition.lat,
    },
    lng: {
      default: mapPosition.lng,
    },
    sensor: {
      type: String,
    },
  },
  components: {
    Header,
    Map,
    Measures,
    ColorfulScale,
    Footer,
    SensorPopup,
    Loader,
    MessagePopup,
  },
  data() {
    return {
      providerReady: false,
      point: null,
      points: {},
      status: "online",
      canHistory: false,
      city: "",
      isShowInfo: false,
      providerObj: null,
    };
  },
  computed: {
    isLoader() {
      return this.provider === "ipfs" && Object.keys(this.points).length === 0;
    },
  },
  mounted() {
    if (this.provider === "remote") {
      this.providerObj = new providers.Remote(config.REMOTE_PROVIDER);
    } else {
      this.providerObj = new providers.Ipfs(config.IPFS);
    }
    this.providerObj.ready().then(() => {
      this.providerReady = true;
      this.providerObj.watch(this.handlerNewPoint);
    });
    if (this.provider === "remote") {
      const iRemote = setInterval(() => {
        if (
          this.providerObj &&
          this.providerObj.connection &&
          markers.isReadyLayers()
        ) {
          clearInterval(iRemote);
          this.canHistory = true;
        }
      }, 1000);
    }
  },
  watch: {
    point(_, oldValue) {
      if (oldValue) {
        markers.hidePath(oldValue.sensor_id);
      }
    },
  },
  methods: {
    async handlerHistory({ start, end }) {
      this.status = "history";
      this.providerObj.watch(null);
      this.handlerClose();
      markers.clear();
      this.providerObj.setStartDate(start);
      this.providerObj.setEndDate(end);
      const sensors = await this.providerObj.lastValuesForPeriod(start, end);
      for (const sensor in sensors) {
        for (const item of sensors[sensor]) {
          this.handlerNewPoint(item);
        }
      }
      const messages = await this.providerObj.messagesForPeriod(start, end);
      for (const message in messages) {
        this.handlerNewPoint(messages[message]);
      }
    },
    handlerNewPoint(point) {
      if (!point.model || !markers.isReadyLayers()) {
        return;
      }
      point.data = point.data
        ? Object.fromEntries(
            Object.entries(point.data).map(([k, v]) => [k.toLowerCase(), v])
          )
        : {};
      markers.addPoint({
        ...point,
        isEmpty: !point.data[this.type.toLowerCase()],
        value: point.data[this.type.toLowerCase()],
      });

      if (point.sensor_id === this.sensor) {
        this.handlerClick(point);
      }

      if (this.point && this.point.sensor_id === point.sensor_id) {
        this.point.log = [
          ...this.point.log,
          {
            data: point.data,
            timestamp: point.timestamp,
          },
        ];
      }

      if (
        Object.prototype.hasOwnProperty.call(
          point.data,
          this.type.toLowerCase()
        ) ||
        Object.prototype.hasOwnProperty.call(point.data, "message")
      ) {
        this.points[point.sensor_id] = point.data;
      }
    },
    async handlerClick(point) {
      let log;
      if (this.status === "history") {
        log = await this.providerObj.getHistoryPeriodBySensor(
          point.sensor_id,
          this.providerObj.start,
          this.providerObj.end
        );
      } else {
        log = await this.providerObj.getHistoryBySensor(point.sensor_id);
      }

      const address = await getAddressByPos(
        point.geo.lat,
        point.geo.lng,
        this.$i18n.locale
      );

      this.point = {
        ...point,
        address,
        log: [...log],
      };
    },
    handlerClose() {
      if (this.point) {
        markers.hidePath(this.point.sensor_id);
      }
      this.point = null;
      instanceMap().setActiveArea({
        position: "absolute",
        top: "0px",
        left: "0px",
        right: "0px",
        height: "100%",
      });
    },
    handlerCloseInfo() {
      this.isShowInfo = false;
    },
    handlerModal(modal) {
      if (modal === "info") {
        this.isShowInfo = true;
      }
    },
    handlerChangeCity(city) {
      this.city = city;
    },
  },
};
</script>

<style>
.sensors-panel--bottom {
  position: relative;
  z-index: 10;
}
</style>
