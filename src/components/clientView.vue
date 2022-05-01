<template>
  <div style="height: 600px; width: 100%">
    <l-map
      v-if="showMap"
      @dblclick="onMapClick"
      :zoom="zoom"
      :center="[
        userLocation.lat || defaultLocation.lat,
        userLocation.lng || defaultLocation.lng,
      ]"
      :options="mapOptions"
      style="height: 80%"
    >
      <l-tile-layer :url="url" :attribution="attribution" />
      <l-marker :lat-lng.sync="marker">
        <l-tooltip
          :content="tooltipContentStatic"
          :options="{ permanent: true }"
        />
      </l-marker>
      <l-marker
        v-if="position.lat && position.lng"
        visible
        draggable
        :icon="icon"
        :lat-lng.sync="position"
        @dragstart="dragging = true"
        @dragend="dragging = false"
      >
        <l-tooltip :content="tooltipContent" :options="{ permanent: true }" />
      </l-marker>
    </l-map>
    <label for="latitude">Latitude:</label>
    <input
      id="latitude"
      v-model="latitudeInput"
      name="latitude"
      type="number"
    />
    <br />
    <label for="longitude">Longitude:</label>
    <input
      id="longitude"
      v-model="longitudeInput"
      name="longitude"
      type="number"
    />
    <br />
    <label>Date:</label>
    <date-pick v-model="date" :displayFormat="'DD.MM.YYYY'"></date-pick>
    <button @click="getPosition">Submit</button>
    <button @click="getSunRiseAndSunSet">Calculate</button>
    <p>
      Sunrise: {{ sunriseStr }}, Sunset: {{ sunsetStr }}, Length of the night:
      {{ night }}
    </p>
  </div>
</template>

<script>
//took scripts from this site for example: https://vue2-leaflet.netlify.app/components/LMarker.html#demo, https://vue2-leaflet.netlify.app/quickstart/#nuxt
import DatePick from "vue-date-pick";
import { Icon, latLng } from "leaflet";
import { LMap, LTileLayer, LMarker, LTooltip } from "vue2-leaflet";
import SunCalc from "suncalc";
delete Icon.Default.prototype._getIconUrl;
Icon.Default.mergeOptions({
  iconRetinaUrl: require("leaflet/dist/images/marker-icon-2x.png"),
  iconUrl: require("leaflet/dist/images/marker-icon.png"),
  shadowUrl: require("leaflet/dist/images/marker-shadow.png"),
});

export default {
  components: {
    DatePick,
    LMap,
    LTileLayer,
    LMarker,
    LTooltip,
  },
  props: {
    value: {
      type: Object,
      required: true,
    },
    defaultLocation: {
      type: Object,
      default: () => ({
        lat: 59.436962,
        lng: 24.753574,
      }),
    },
  },
  //Here we define the data so that it can be used and here we can see some map variables (default values etc)
  //https://medium.com/swlh/create-an-interactive-location-selector-with-vue-js-and-leaflet-5808c55b4636 here i saw this interesting location selector
  data() {
    return {
      latitudeInput: 59.436962,
      longitudeInput: 24.753574,
      sunriseStr: "",
      sunsetStr: "",
      night: "",
      date: "2019-01-01",
      loading: false,
      userLocation: {},
      marker: latLng(this.latitudeInput, this.longitudeInput),
      zoom: 9,
      url: "https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png",
      attribution:
        '&copy; <a href="http://osm.org/copyright">OpenStreetMap</a> contributors',
      position: {},
      address: "",
      mapOptions: {
        zoomSnap: 0.5,
      },
      showMap: true,
    };
  },
  //Every time a website is loaded, these methods are running
  mounted() {
    this.getUserPosition();
    this.getPosition();
    this.getSunRiseAndSunSet();
  },
  watch: {
    position: {
      deep: true,
      async handler(value) {
        this.address = await this.getAddress();
        this.$emit("input", { position: value, address: this.address });
      },
    },
    marker: {
      deep: true,
      async handler(marker) {
        this.address = await this.getAddress();
        this.$emit("input", { marker: marker, address: this.address });
      },
    },
  },
  computed: {
    //These methods show the text box next to the marker, which is what we see on the map
    tooltipContent() {
      if (this.dragging) return "...";
      if (this.loading) return "Loading...";
      return `<strong>${this.address.replace(
        ",",
        "<br/>"
      )}</strong> <hr/><strong>lat:</strong> ${
        this.position.lat
      }<br/> <strong>lng:</strong> ${this.position.lng}`;
    },
    tooltipContentStatic() {
      if (this.dragging) return "...";
      if (this.loading) return "Loading...";
      return `<strong>${this.address.replace(
        ",",
        "<br/>"
      )}</strong> <hr/><strong>lat:</strong> ${
        this.marker.lat
      }<br/> <strong>lng:</strong> ${this.marker.lng}`;
    },
  },
  methods: {
    //Initially, the address is set to an unresolved address, then the address is tried to be retrieved from the openstreet folder, and if the status is 200, the address is returned.
    async getAddress() {
      this.loading = true;
      let address = "Unresolved address";
      try {
        const { lat, lng } = this.position;
        const result = await fetch(
          `https://nominatim.openstreetmap.org/reverse?format=jsonv2&lat=${lat}&lon=${lng}`
        );
        if (result.status === 200) {
          const body = await result.json();
          address = body.display_name;
        }
      } catch (e) {
        console.error("Reverse Geocode Error->", e);
      }
      this.loading = false;
      return address;
    },
    onMapClick(value) {
      this.position = value.latlng;
    },
    //Getting input coordinates position
    getPosition() {
      this.marker = {
        lat: this.latitudeInput,
        lng: this.longitudeInput,
      };
    },
    //Here we calculate sunrise and sunset using suncalc functions, also converting minutes to hours
    getSunRiseAndSunSet() {
      var times = SunCalc.getTimes(
        new Date(),
        this.latitudeInput,
        this.longitudeInput
      );
      (this.sunriseStr =
        times.sunrise.getHours() + times.sunrise.getMinutes() / 60),
        (this.sunsetStr =
          times.sunset.getHours() + times.sunset.getMinutes() / 60),
        (this.night = 24 - (this.sunsetStr - this.sunriseStr));
    },
    //Getting user position on the map
    async getUserPosition() {
      if (navigator.geolocation) {
        navigator.geolocation.getCurrentPosition((pos) => {
          this.userLocation = {
            lat: pos.coords.latitude,
            lng: pos.coords.longitude,
          };
        });
      }
    },
  },
};
</script>
