<template>
  <div>
    <MapFilterComponent />
    <div style="z-index:1;" id='mapContainer'></div>
  </div>
</template>

<script>
import MapFilterComponent from './MapFilterComponent.vue'
import mapboxgl from 'mapbox-gl';
import dayjs from 'dayjs';
import customParseFormat from 'dayjs/plugin/customParseFormat';
import { getPopupHtmlString, legendColorMap } from '../mapUtils'

dayjs.extend(customParseFormat)


export default {
  name: 'Map',
  components: {
    MapFilterComponent
  },
  data() {
    return {
      mapboxKey: process.env.VUE_APP_MAPBOX_KEY,
      mapView: null,
    }
  },
  created() {
    this.loadSightings()
  },
  updated() {
    this.loadSightings()
  },
  methods: {
    mapSightings() {
      // Grab access token for Mapbox
      mapboxgl.accessToken = this.mapboxKey

      //TODO: use styles on the map????

      // Initialise mapbox container
      const map = new mapboxgl.Map({
        container: 'mapContainer',
        style: 'mapbox://styles/mapbox/light-v10',
        center: [-122.312490, 47.951812],
        zoom: 7.0
      });


      // Add navigation buttons on top right of the map
      const nav = new mapboxgl.NavigationControl()
      map.addControl(nav, "top-right")

      // Cache map
      this.mapView = map
      // Resize map to fit into screen width
      this.mapView.resize()

      let geoData = {
        "type": "FeatureCollection",
        "features": this.filteredSightings
      }

      const currentPage = this.getParent;

      // Function to generate the mapboxgl match expression
      function generateMatchExpression(colorMappings) {
        const matchExpression = ['match', ['get', 'type']];
        for (const [type, color] of Object.entries(colorMappings)) {
          matchExpression.push(type, color);
        }
        matchExpression.push('#000000'); // Set default to black if type doesn't match any
        return matchExpression;
      }

      // On load event
      map.on('load', function () {
        if (currentPage === 'Heatmap') {
          map.addLayer({
            id: 'ssemmi-heat-layer',
            type: 'heatmap',
            source: {
              type: 'geojson',
              data: geoData
            },
            minZoom: 0,
            paint: {
              // increase weight as diameter breast height increases
              'heatmap-weight': [
                'interpolate',
                ['linear'],
                ['get', 'no_sighted'],
                1,
                2,
                3,
                4
              ],
              // Increase the heatmap color weight weight by zoom level
              // heatmap-intensity is a multiplier on top of heatmap-weight
              'heatmap-intensity': [
                'interpolate',
                ['linear'],
                ['zoom'],
                3,
                1,
                4,
                3
              ],
              // Color ramp for heatmap.  Domain is 0 (low) to 1 (high).
              // Begin color ramp at 0-stop with a 0-transparancy color
              // to create a blur-like effect.
              'heatmap-color': [
                'interpolate',
                ['linear'],
                ['heatmap-density'],
                0,
                'rgba(33,102,172,.1)',
                0.2,
                'rgb(103,169,207)',
                0.4,
                'rgb(209,229,240)',
                0.6,
                'rgb(253,219,199)',
                0.8,
                'rgb(233,156,119)',
                1,
                'rgb(227,67,86)'
              ],
              // Adjust the heatmap radius by zoom level
              'heatmap-radius': [
                'interpolate',
                ['linear'],
                ['zoom'],
                10,
                20,
                30,
                50
              ],
              // Transition from heatmap to circle layer by zoom level
              'heatmap-opacity': [
                'interpolate',
                ['linear'],
                ['zoom'],
                6,
                1,
                9,
                1
              ]
            }
          }
          )
        } else {
          map.addLayer({
            id: 'ssemmi-map-layer',
            type: 'circle',
            source: {
              type: 'geojson',
              data: geoData,
            },
            paint: {
              // Set size of circle pinpoint based on how large the sighting is (all made same size for now, was 0 4 5 12)
              'circle-radius': [
                'interpolate',
                ['linear'], ['zoom'],
                6, 6,
                8, 10,
                10, 14
              ],
              // Set colour of circle pinpoint based of number of sightings
              'circle-opacity': 0.9,
              'circle-color': generateMatchExpression(legendColorMap)
            }
          })
        }
      })




      // Click listener to display extra information upon clicking on a sightings point
      map.on('click', 'ssemmi-map-layer', function (e) {
        let coordinates = e.features[0].geometry.coordinates.slice();

        // Ensure that if the map is zoomed out such that multiple
        // copies of the feature are visible, the popup appears
        // over the copy being pointed to.
        while (Math.abs(e.lngLat.lng - coordinates[0]) > 180) {
          coordinates[0] += e.lngLat.lng > coordinates[0] ? 360 : -360;
        }

        new mapboxgl.Popup({ closeButton: true })
          .setLngLat(coordinates)
          .setHTML(getPopupHtmlString(e.features[0].properties))
          .addTo(map);
      })

      //Save reference to the map for rerendering
      this.$store.commit("setMap", map)
    },
    loadSightings() {
      // Call the method to retreive data
      let dataPoints = []
      let speciesList = new Set()
      let contributorList = new Set()

      this.$store.dispatch("get_sightings")
        .then((currSights) => {
          Object.values(currSights).forEach((value) => {
            //get curr date
            // Create new array instance of two numbers for mapbox marker coordinate
            if (value && value.created) {
              // Check if the fields for the sighting is valid or compatible
              let filtered_long = (isNaN(value.longitude)) ? 1 : value.longitude
              let filtered_lat = (isNaN(value.latitude)) ? 1 : value.latitude
              let filtered_sightings = (isNaN(value.no_sighted)) ? 1 : value.no_sighted
              let filtered_date = dayjs('2011-01-01 20:00:00')
              let days_ago = 0
              let f_day = 1
              let f_month = 1
              let f_year = 2011
              let f_epoch_date = new Date().getTime()

              if (filtered_date.isValid()) {
                filtered_date = dayjs(value.created.substr(0, 10).split(' ')[0], ['YYYY-MM-DD', 'MM/DD/YY', 'DD/MM/YY', 'D/M/YY'])
                f_day = filtered_date.date()
                f_month = filtered_date.month() + 1
                f_year = filtered_date.year()
                days_ago = dayjs().diff(filtered_date, 'day')
                f_epoch_date = new Date(filtered_date).getTime()
              }

              const sightingEntry = {
                "type": "Feature",
                "geometry": {
                  "type": "Point",
                  "coordinates": [filtered_long, filtered_lat]
                },
                "properties": {
                  "entity": value.data_source_entity,
                  "ssemmi_id": value.ssemmi_id,
                  "created": value.created,
                  "type": value.type,
                  "day": f_day,
                  "month": f_month,
                  "year": f_year,
                  "epoch_date": f_epoch_date,
                  "no_sighted": filtered_sightings,
                  "days_ago": days_ago.toString(),
                  "witness": value.data_source_witness,
                  "comments": value.data_source_comments,
                  "ssemmi_date_added": value.ssemmi_date_added,
                  "verified": value.trusted,
                  "sightingRef": value.entry_id,
                }
              }

              dataPoints.push(sightingEntry)
            }
          })

          //Sort data chronologically. Required as filtering by date uses a sliding window.
          dataPoints.sort((a, b) => {
            if (dayjs(a.properties.created.slice(0, 19)).isBefore(dayjs(b.properties.created.slice(0, 19)))) {
              return -1
            } else {
              return 1
            }
          })

          //Collect species and contributors
          dataPoints.forEach(sighting => {
            speciesList.add(sighting.properties.type)
            contributorList.add(sighting.properties.witness)
          })

          //Sort species and contributors alphabetically, disregarding case
          speciesList = [...speciesList]
          speciesList = speciesList.filter(species => {
            if (!species) {
              return false
            } else {
              return true
            }
          }).sort((a, b) => a.toLowerCase().localeCompare(b.toLowerCase()));

          contributorList = [...contributorList]
          contributorList = contributorList.filter(contributor => {
            if (!contributor) {
              return false
            } else {
              return true
            }
          }).sort((a, b) => a.toLowerCase().localeCompare(b.toLowerCase()))

          //Store sightings and filter options in store
          this.$store.commit("setMapOptions", {
            contributors: contributorList,
            species: speciesList
          })

          this.$store.commit("setSightings", dataPoints)

          //Apply default filters on first render.
          //Reduces initial page load by only mapping previous 7 days of data.
          this.$store.commit("applyMapFilters")

          this.mapSightings()
          this.$store.commit("setActiveMapLayer", this.getActiveMapLayer)
        })
    },
  },
  computed: {
    isAuth() {
      return this.$store.state.isAuthenticated
    },
    getParent() {
      return this.$parent.$options.name
    },
    getActiveMapLayer() {
      let activeComponent = this.getParent

      if (activeComponent === "Visualiser") {
        return "ssemmi-map-layer"
      } else if (activeComponent === "Heatmap") {
        return "ssemmi-heat-layer"
      } else if (activeComponent === "Home") {
        return "ssemmi-map-layer"
      } else {
        console.log("Map rendered on incorrect page")
        return ""
      }
    },
    filteredSightings() {
      return this.$store.getters.getFilteredSightings
    }
  }
}
</script>

<style>
#mapContainer {
  width: 100%;
  height: 100%;
  margin: 0 auto;
  position: relative;
}

#active-date {
  font-weight: bold;
}

#widget {
  position: fixed;
  width: 30%;
  top: 6vh;
  left: 1vh;
  margin: 10px;
  padding: 10px 20px;
  background-color: white;
  position: absolute;
}

.slider-class {
  margin-left: 5%;
  font-size: 1rem;
}

.widget-row {
  height: 12px;
  width: 100%;
}

.colors {
  background: linear-gradient(to right, #bb0314, #bd2e2e, #e70ca9, #9a0fce, #0f86dc, #1f39a7);
}

.label {
  width: 15%;
  display: inline-block;
  text-align: center;
}
</style>
