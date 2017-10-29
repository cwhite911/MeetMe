<template>
  <div id='map'></div>
</template>
<script>
const mapboxgl = require('mapbox-gl')
const MapboxGeocoder = require('mapbox-gl-geocoder')
const polyline = require('@mapbox/polyline')

// const MapboxDirections = require('@mapbox/mapbox-gl-directions')
const turf = require('@turf/turf')
console.log('mapbox-gl:', mapboxgl)
console.log('MapboxDirections:', window.MapboxDirections)
console.log('turf:', turf)
export default {
  name: 'MeetMap',
  mounted () {
    console.log('MeetMap: ready():', this)
    this.createMap()
  },
  data () {
    return {
      departureData: {
        type: 'geojson',
        data: {
          type: 'FeatureCollection',
          features: []
        }
      },
      destinationData: {
        type: 'geojson',
        data: {
          type: 'FeatureCollection',
          features: []
        }
      },
      routesData: {
        type: 'geojson',
        data: {
          type: 'FeatureCollection',
          features: []
        }
      }
    }
  },
  methods: {
    createMap () {
      // let vm = this
      mapboxgl.accessToken = 'pk.eyJ1IjoiY3R3aGl0ZSIsImEiOiItb0dqdUlZIn0.4Zb1DGESXnx0ePxMVLihZQ'
      this.map = new mapboxgl.Map({
        container: 'map',
        style: 'mapbox://styles/mapbox/streets-v9',
        center: [-96, 37.8],
        zoom: 3
      })
      this.map.addControl(new mapboxgl.NavigationControl())

      this.geocoder = new MapboxGeocoder({
        accessToken: mapboxgl.accessToken
      })

      // let geocoder = this.geocoder
      this.directions = new window.MapboxDirections({
        accessToken: mapboxgl.accessToken,
        // unit: 'metric',
        profile: 'cycling',
        controls: {
          inputs: false,
          instructions: false
        }
        // geocoder: geocoder
      })
      this.map.addControl(this.directions)

      this.map.addControl(this.geocoder, 'top-left')
      console.log('MapView: createMap:', this.map)

      this.map.on('load', () => {
        this.map.addSource('departures', this.departureData)

        // Departures Layer
        this.map.addLayer({
          'id': 'departures',
          'source': 'departures',
          'type': 'circle',
          'paint': {
            'circle-radius': 10,
            'circle-color': '#007cbf'
          }
        })

        this.map.addSource('destination', this.destinationData)

        // Destination Layer
        this.map.addLayer({
          'id': 'destination',
          'source': 'destination',
          'type': 'circle',
          'paint': {
            'circle-radius': 10,
            'circle-color': '#007'
          }
        })

        this.map.addSource('routes', this.routesData)

        // Routes Layer
        this.map.addLayer({
          'id': 'routes',
          'source': 'routes',
          'type': 'line',
          layout: {
            'line-join': 'round',
            'line-cap': 'round'
          },
          paint: {
            'line-color': '#888fff',
            'line-width': 8
          }
        })

        this.directions.on('route', ev => {
          // console.log('The route changed:', ev, ev.route[0], polyline)
          let route = polyline.toGeoJSON(ev.route[0].geometry)
          // console.log('Route', route)
          this.routesData.data.features.push(route)
          this.map.getSource('routes').setData(this.routesData.data)
        })

        this.directions.on('error', error => {
          console.log('directions error:', error)
        })
        // Listen for the `geocoder.input` event that is triggered when a user
        // makes a selection and add a symbol that matches the result.
        this.geocoder.on('result', (ev) => {
          // Push the geocoded point to the departure data feature and set it on the source
          this.departureData.data.features.push(ev.result)
          this.map.getSource('departures').setData(this.departureData.data)
          // Create bounding box to change update the map view
          // let featureCollection = turf.featureCollection(this.departureData.data.features)
          let bbox = turf.bbox(turf.featureCollection(this.departureData.data.features))
          this.map.fitBounds(bbox, {padding: 50, maxZoom: 10})
          console.log('the length', this.departureData.data.features.length)
          if (this.departureData.data.features.length > 1) {
            // Add the updated destination
            let center = turf.center(turf.featureCollection(this.departureData.data.features))
            console.log('center', center.geometry.coordinates)
            this.destinationData.data.features = [center]
            this.map.getSource('destination').setData(this.destinationData.data)

            console.log(this.directions)
            // var v1 = new mapboxgl.LngLat(this.departureData.data.features[0].geometry.coordinates[0], this.departureData.data.features[0].geometry.coordinates[1])
            // var v2 = new mapboxgl.LngLat(this.destinationData.data.features[0].geometry.coordinates[0], this.destinationData.data.features[0].geometry.coordinates[1])
            // console.log('v1', v1, v2)
            this.departureData.data.features.forEach(feature => {
              // this.directions = new window.MapboxDirections({
              //   accessToken: mapboxgl.accessToken,
              //   // unit: 'metric',
              //   profile: 'cycling',
              //   controls: {
              //     inputs: false,
              //     instructions: false
              //   }
              //   // geocoder: geocoder
              // })
              // this.map.addControl(this.directions)
              this.directions.setOrigin(feature.geometry.coordinates)
              this.directions.setDestination(this.destinationData.data.features[0].geometry.coordinates)
              // this.directions.actions.setProfile('driving')
            })
            // this.directions.setOrigin(this.departureData.data.features[0].geometry.coordinates)
            // this.directions.setDestination(this.destinationData.data.features[0].geometry.coordinates)
            // this.directions.actions.setProfile('driving')
          }
        })
      })
    }
  }

}
</script>

<!-- Add 'scoped' attribute to limit CSS to this component only -->
<style scoped>

html body { margin:0; padding:0; }
#map {
  position:absolute;
  top:0;
  bottom:0;
  width:100%;
}
</style>
