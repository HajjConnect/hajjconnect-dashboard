<template>
<div class="container">
  <!-- <img src="../assets/logo.png" alt="HajjConnect"> -->
  <h1>HajjConnect - Operator Dashboard</h1>
  <sui-statistics-group :columns="3">
    <sui-statistic in-group>
      <sui-statistic-value>{{ nLost }}</sui-statistic-value>
      <sui-statistic-label>Lost</sui-statistic-label>
    </sui-statistic>
    <sui-statistic in-group>
      <sui-statistic-value>{{ nRisk }}</sui-statistic-value>
      <sui-statistic-label>At Risk</sui-statistic-label>
    </sui-statistic>
    <sui-statistic in-group>
      <sui-statistic-value>{{ nRecover }}</sui-statistic-value>
      <sui-statistic-label>Recovering</sui-statistic-label>
    </sui-statistic>
  </sui-statistics-group>
  <GmapMap
    :center="center"
    :zoom="7"
    map-type-id="terrain"
    style="width: 500px; height: 300px"
  >
      <gmap-info-window :options="infoOptions" :position="infoWindowPos" :opened="infoWinOpen" @closeclick="infoWinOpen=false">
        {{infoContent}}
      </gmap-info-window>
    <GmapMarker
      :key="index"
      v-for="(m, index) in getPersons"
      :position="m.position"
      :clickable="true"
      :draggable="false"
      @click="toggleInfoWindow(m,index)"
    />
  </GmapMap>
  <sui-card-group :items-per-row="4" class="doubling" v-if="getPersons">
    <sui-card v-for="(person, index) in getPersons" :key="'person-' + index" :class="person.regionColor" >
      <sui-dimmer-dimmable
        @mouseenter.native="cardActive = index"
        @mouseleave.native="cardActive = null">
        <sui-image v-if="person.image" :src="person.image" size="medium"/>
        <sui-dimmer blurring :active="cardActive === index">
          <sui-button icon="crosshairs">Locate</sui-button>
        </sui-dimmer>
      </sui-dimmer-dimmable>
      <sui-card-content>
        <sui-card-header>{{ person.name }}</sui-card-header>
        <sui-card-meta>{{ person.gender }}, {{ person.country }}</sui-card-meta>
        <sui-card-description v-if="person.description">{{ person.description }}</sui-card-description>
      </sui-card-content>
      <sui-card-content extra >
        <sui-icon name="marker" />{{ person.status }}
      </sui-card-content>
    </sui-card>
  </sui-card-group>
  <sui-segment v-if="getPersons.length === 0">No match found</sui-segment>
</div>
</template>

<script>
import { rows } from 'google-spreadsheets'
import { promisify } from 'bluebird'

const getRows = promisify(rows)
const sheetId = '1beMhjygRYfRXI3Xi0QP7g5kExmrc8CzrivIdw0I6LUo'

export default {
  data () {
    return {
      persons: [],
      isLoaded: false,
      cardActive: null,
      search: '',
      sort: null,
      role: null,
      options: [
        { text: 'City alphabetical', value: 'city' },
        { text: 'Ladies first', value: 'female' }
      ],
      center: {
        lat: 47.376332,
        lng: 8.547511
      },
      infoContent: '',
      infoWindowPos: null,
      infoWinOpen: false,
      currentMidx: null,
      //optional: offset infowindow so it visually sits nicely on top of our marker
      infoOptions: {
        pixelOffset: {
          width: 0,
          height: -35
        }
      },
    }
  },
  mounted () {
    getRows({
      key: sheetId,
      worksheet: 1
    }).then(res => {
      this.persons = res.map(item => {
        return {
          ...item, 
          position: {
            lng: Number(item.longitude),
            lat: Number(item.latitude)
          }
        }
      })
      this.isLoaded = true
    })
  },
  computed: {
    nLost () {
      return this.persons.map(item => item.status).filter(value => value === 'lost').length
    },
    nRisk () {
      return this.persons.map(item => item.status).filter(value => value === 'at risk').length
    },
    nRecover () {
      return this.persons.map(item => item.status).filter(value => value === 'recovering').length
    },
    getPersons () {
      let persons = this.persons.filter((person) => {
        return Object.values(person).join(' ').toLowerCase().includes(this.search.toLowerCase())
      })

      switch (this.sort) {
        case 'female':
          persons = persons.sort(function (a, b) {
            if (a.gender === 'female') return -1
            if (b.gender === 'male') return 1
            return 0
          })
          break

        default:
          persons = persons.sort(function (a, b) {
            if (a.city < b.city) return -1
            if (a.city > b.city) return 1
            return 0
          })
      }

      return persons.filter(person => {
        if (this.role === null) return true
        return person.roles.includes(this.role)
      })
    }
  },
  methods: {
    toggleInfoWindow (marker, idx) {
      this.infoWindowPos = marker.position;
      this.infoContent = marker.name;

      //check if its the same marker that was selected if yes toggle
      if (this.currentMidx == idx) {
        this.infoWinOpen = !this.infoWinOpen;
      }
      //if different marker set infowindow to open and reset current marker index
      else {
        this.infoWinOpen = true;
        this.currentMidx = idx;
      }
    }
  }
}
</script>
