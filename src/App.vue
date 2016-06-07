<template>
  <div id="app">
    <button @click="setCenter()">
      我的位置：{{ position.lat }}, {{ position.lng }}
    </button>
    <button @click="toggleMenu()">座標列表</button>
    <ul :class="{ 'menu': hideMenu }">
      <li v-for="m in markers">
        <b v-if="m.key === userId">4ni </b><a @click="setCenter(m)">{{ m.name }}</a> [<a @click="track(m)">track</a>]
      </li>
    </ul>

    <map
      :center.sync="center"
      :map-type-id.sync="terrain"
      :zoom.sync="18">
      <marker
        v-for="m in markers"
        :position.sync="m.position"
        :label.sync="m.name">
      </marker>
    </map>

  </div>
</template>

<script>
import firebase from 'firebase'
import {
  load,
  Map,
  Marker,
} from 'vue-google-maps'

import moment from 'moment'
import Chance from 'chance'

function generateMapId () {
  return db.ref('/maps/').push().key
}

function generateUserId (mapId) {
  return db.ref('/maps/' + mapId).push().key
}

let config = {
  apiKey: "AIzaSyBViACd0PRoj3M33fMxpVs7PegSdSz3ofg",
  authDomain: "fire-geo-bdc76.firebaseapp.com",
  databaseURL: "https://fire-geo-bdc76.firebaseio.com",
  storageBucket: "fire-geo-bdc76.appspot.com",
};

load('AIzaSyBViACd0PRoj3M33fMxpVs7PegSdSz3ofg')

firebase.initializeApp(config);

const db = firebase.database()

export default {
  components: {
    Map, Marker
  },
  data () {
    return {
      mapId: '',
      userId: '',
      center: { lat: 0, lng: 0 },
      position: { lat: 0, lng: 0 },
      markers: [],
      hideMenu: true,
      tracked: '',
    }
  },

  ready () {
    this.mapId = location.hash.slice(1)
    if (this.mapId === '') {
      this.mapId = generateMapId()
      location.hash = this.mapId
    }

    this.userId = localStorage.getItem('fire-geo-userId')
    if (!this.userId) {
      this.userId = generateUserId(this.mapId)
      localStorage.setItem('fire-geo-userId', this.userId)
    }
    this.name = (new Chance()).first()

    let ref = db.ref('/maps/' + this.mapId)

    ref.on('child_added', (data) => {
      this.markers.push({
        key: data.key,
        name: data.val().name,
        position: data.val().position
      })
    })

    ref.on('child_changed', (data) => {
      let marker = this.markers.find((m) => m.key === data.key)
      marker.name = data.val().name
      marker.position = data.val().position
    })

    ref.on('child_removed', (data) => {
      this.markers = this.markers.filter((m) => m.key !== data.key)
    })

    let init = false
    navigator.geolocation.watchPosition((position) => {
      this.position.lat = position.coords.latitude
      this.position.lng = position.coords.longitude

      ref.child(this.userId).set({
        name: this.name,
        position: this.position,
        timestamp: Date.now()
      })

      if (!init) {
        this.setCenter()
        init = true
      }
    }, (err) => {
      alert('Failed to watch your position')
    }, {
      enableHighAccuracy: true
    })

    setInterval(() => {
      ref.once('value', (markers) => {
        const now = moment(Date.now())

        markers.forEach((m) => {
          const timestamp = moment(m.val().timestamp)
          if (timestamp.add(10, 'minutes').isBefore(now)) {
            ref.child(m.key).remove()
          }
        })
      })
    }, 5000)

  },

  methods: {
    setCenter (m) {
      if (m) {
        this.center = m.position
      } else {
        this.center = this.position
        db.ref(`/maps/${this.mapId}/${this.tracked}/`).off('value')
        this.tracked = ''
      }
    },

    toggleMenu () {
      this.hideMenu = !this.hideMenu
    },

    track (m) {
      this.tracked = m.key
      db.ref(`/maps/${this.mapId}/${m.key}`).on('value', (data) => {
        this.setCenter(data.val())
      })
    }
  },

}
</script>

<style>
body {
  margin: 0px;
  padding: 0px;
}

map {
  width: 100%;
  height: 450px;
  display: block;
}

.menu {
  display: none;
}

a {
  cursor: pointer;
  text-decoration: underline;
}

</style>
