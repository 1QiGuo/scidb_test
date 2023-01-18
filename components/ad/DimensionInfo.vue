<template>
  <v-hover v-slot:default="{ hover }" open-delay="0">
    <v-card
      class="mx-auto"
      :elevation="hover ? 6 : 2"
      :class="{ 'on-hover': hover }"
    >
      <v-card-title class="primary white--text title-1"
        >Cell clustering
      </v-card-title>
      <v-card-text>
        <p class="my-3 text--primary"></p>
        <p class="display-1 text--primary"></p>
        <v-row>
          <v-col xs="12" md="8" lg="3" class="px-4 py-0 my-0">
            <p class="subtitle-1 font-weight-bold text--primary">
              Cell type & subcluster:<v-tooltip top>
                <template v-slot:activator="{ on, attrs }">
                  <v-icon color="primary" dark v-bind="attrs" v-on="on"
                    >mdi-help-circle-outline</v-icon
                  >
                </template>
                <span
                  >Select UMAP coordinates source, either using all cells or
                  subset of a specific cell type.</span
                >
              </v-tooltip>
            </p>

            <v-autocomplete
              v-model="clusterCoordinatesSelect"
              :items="clusterCoordinatesItems"
              item-text="cell_type"
              item-value="cell_type"
              label=""
              return-object
              lg="3"
              @change="updateCluster()"
            ></v-autocomplete>
            <p
              v-if="
                clusterCoordinatesSelect === 'All cell types' &&
                  dataset[0].ari_score !== 'NA'
              "
              class="text--primary"
            >
              ARI score: {{ dataset[0].ari_score }}
              <v-tooltip top>
                <template v-slot:activator="{ on, attrs }">
                  <v-icon color="primary" dark v-bind="attrs" v-on="on"
                    >mdi-help-circle-outline</v-icon
                  >
                </template>
                <span
                  >The identified cell labels are evaluated by the Adjusted Rand
                  Index (ARI) by comparing with the benchmark labels provided
                  from the original study.</span
                >
              </v-tooltip>
            </p>
            <p
              v-if="
                clusterCoordinatesSelect === 'All cell types' &&
                  dataset[0].silhouette_score > 0.4
              "
              class="text--primary"
            >
              Average silhouette width: {{ dataset[0].silhouette_score }}
              <v-tooltip top>
                <template v-slot:activator="{ on, attrs }">
                  <v-icon color="primary" dark v-bind="attrs" v-on="on"
                    >mdi-help-circle-outline</v-icon
                  >
                </template>
                <span
                  >Silhouette width measures how similar a cell is to its type
                  compared to other clusters.</span
                >
              </v-tooltip>
            </p>
          </v-col>
          <v-col xs="12" md="8" lg="3" class="px-4 py-0 my-0"> </v-col>
          <v-col xs="12" md="8" lg="3" class="px-4 py-0 my-0">
            <p class="subtitle-1 font-weight-bold text--primary">
              Gene expression<v-tooltip top>
                <template v-slot:activator="{ on, attrs }">
                  <v-icon color="primary" dark v-bind="attrs" v-on="on"
                    >mdi-help-circle-outline</v-icon
                  >
                </template>
                <span
                  >Select or search gene of interest to plot gene expression
                  value on the UMAP plot, darker points indicate higher
                  expression values.</span
                >
              </v-tooltip>
            </p>
            <v-autocomplete
              v-model="gene"
              :items="allGenes"
              :label="'Select or search: (' + allGenes.length + ' genes)'"
              @change="updateExpression()"
            ></v-autocomplete>
          </v-col>
        </v-row>
        <v-row>
          <v-col xs="12" md="6" lg="3" class="px-4 py-0 my-0">
            <p class="subtitle-1 font-weight-bold">
              Point size:
              <v-tooltip top>
                <template v-slot:activator="{ on, attrs }">
                  <v-icon color="primary" dark v-bind="attrs" v-on="on"
                    >mdi-help-circle-outline</v-icon
                  >
                </template>
                <span
                  >Limit testing to genes which show, on average, at least
                  X-fold difference (log-scale) between the two groups of
                  cells.</span
                >
              </v-tooltip>
            </p>
            <v-slider
              v-model="pointSize"
              max="10"
              min="1"
              hide-details
              :thumb-size="24"
              thumb-label="always"
              class="align-center"
              step="1"
              ticks="always"
              tick-size="4"
            >
            </v-slider>
          </v-col>
        </v-row>

        <v-row>
          <v-col xs="12" md="12" lg="6" class="px-4 py-0 my-0">
            <div>
              <client-only>
                <vue-plotly
                  :data="allCellDim"
                  :layout="layout"
                  :options="options"
                />
              </client-only>
            </div>
          </v-col>
          <v-col xs="12" md="12" lg="6" class="px-4 py-0 my-0">
            <div v-if="showPlot">
              <client-only>
                <vue-plotly
                  :data="expressionDim"
                  :layout="layout"
                  :options="options"
                />
              </client-only>
            </div>
            <div v-else>
              <p>Please enter a gene symbol.</p>
            </div>
          </v-col>
          <v-col xs="12" md="12" lg="6" class="px-4 py-0 my-0"
            ><dataset-barplot
              source="single-cell"
              :freq="dimensionFreq"
              :colors="allColors"
            ></dataset-barplot>
          </v-col>
          <v-col xs="12" md="12" lg="6" class="px-4 py-0 my-0"
            ><div v-if="showPlot">
              <dataset-violin
                :result="dimensionViolin"
                :colors="allColors"
              ></dataset-violin>
            </div>
          </v-col>
        </v-row>
      </v-card-text>
    </v-card>
  </v-hover>
</template>

<script>
import _ from 'lodash'
import { mapState } from 'vuex'
import DatasetBarplot from '../figures/DimensionBarplot.vue'
import DatasetViolinplot from '../figures/DimensionViolinplot.vue'
export default {
  name: 'DimensionInfo',
  components: {
    'dataset-barplot': DatasetBarplot,
    'dataset-violin': DatasetViolinplot
  },
  props: {
    dataId: {
      type: String,
      required: true
    },
    dataset: {
      type: Array,
      required: true
    },
    dimension: {
      type: Array,
      required: true
    },
    ct: {
      type: Array,
      required: true
    }
  },

  data() {
    return {
      gene: null,
      search: '',
      pointSize: 4,
      headers: [
        { text: 'Gene name', value: 'gene' },
        { text: 'Log fold-change', value: 'avg_logFC' },
        { text: 'Adjusted p-value', value: 'p_val_adj' }
      ],
      clusterCoordinatesSelect: 'All cell types',
      enrich: [],
      layout: {
        autosize: true,
        height: 800,
        margin: {
          l: 10,
          r: 10,
          b: 210,
          t: 20,
          pad: 0
        },
        legend: {
          font: {
            size: 14
          },
          orientation: 'h',
          marker: {
            size: 20
          }
        },
        xaxis: {
          tickfont: {
            size: 16,
            color: 'black'
          },
          title: {
            text: 'UMAP_1',
            font: {
              size: 18
            }
          }
        },
        yaxis: {
          tickfont: {
            size: 16,
            color: 'black'
          },
          title: {
            text: 'UMAP_2',
            font: {
              size: 18
            }
          }
        }
      },
      options: {
        toImageButtonOptions: {
          format: 'png', // one of png, svg, jpeg, webp
          filename: 'download_umap' + new Date().toISOString(),
          height: 1000,
          width: 800,
          scale: 1 // Multiply title/legend/axis/canvas sizes by this factor
        },
        showLink: false,
        displaylogo: false,
        modeBarButtonsToRemove: [
          'hoverCompareCartesian',
          'hoverClosestCartesian'
        ]
      }
    }
  },
  computed: {
    ...mapState({
      expression: (state) => state.ad.expression,
      allGenes: (state) => state.ad.expressionGenes
    }),
    allColors() {
      const result = this.allCellDim.map((i) => {
        return i.marker.color
      })
      return result
    },
    dimensionFreq() {
      const names = this.dimension
        .map((item) => {
          if (item.subcluster !== 'all') {
            return `${item.cell_type}_${item.subcluster}`
          } else {
            if (item.cell_type === 'Oligodendrocyte precursor cells') {
              return 'Oligodendrocyte <br>precursor cells'
            }
            return item.cell_type
          }
        })
        .sort()
      const counts = {}
      for (const num of names) {
        counts[num] = counts[num] ? counts[num] + 1 : 1
      }
      return counts
    },
    showPlot() {
      return this.gene !== null
    },
    dimensionViolin() {
      const rawNames = this.dimension.map((item) => {
        if (item.subcluster !== 'all') {
          return `${item.cell_type}_${item.subcluster}`
        } else {
          if (item.cell_type === 'Oligodendrocyte precursor cells') {
            return 'Oligodendrocyte <br>precursor cells'
          }
          return item.cell_type
        }
      })
      const indices = Array.from(rawNames.keys())
      indices.sort((a, b) => rawNames[a].localeCompare(rawNames[b]))
      const names = indices.map((i) => rawNames[i])
      const expression = indices.map((i) => this.expression[i])
      const geneName = this.gene
      const clusterName = this.clusterCoordinatesSelect
      return { names, expression, geneName, clusterName }
    },
    clusterCoordinatesItems() {
      const cellTypeList = _.map(this.ct, 'cell_type')
      cellTypeList.unshift('All cell types')
      return cellTypeList
    },
    allCellDim() {
      function getTrace(dim, cellType, customColor, markerSize) {
        const currentDimension = dim.filter((row) => row.cell_type === cellType)
        const X = _.map(currentDimension, 'umap_1')
        const Y = _.map(currentDimension, 'umap_2')
        const cellNames = _.map(currentDimension, 'cell_name')
        const trace = {
          x: X,
          y: Y,
          marker: {
            size: markerSize,
            symbol: 'circle',
            color: customColor
          },
          mode: 'markers',
          type: 'scatter',
          name: cellType,
          text: cellNames
        }
        return trace
      }
      function unique(arr) {
        if (!Array.isArray(arr)) {
          console.log('type error!')
          return
        }
        const array = []
        for (let i = 0; i < arr.length; i++) {
          if (!array.includes(arr[i])) {
            array.push(arr[i])
          }
        }
        return array
      }
      function getSubclusterTrace(
        dim,
        select,
        subcluster,
        customColor,
        markerSize
      ) {
        const currentDimension = dim.filter(
          (row) => row.subcluster === subcluster
        )
        const X = _.map(currentDimension, 'umap_1')
        const Y = _.map(currentDimension, 'umap_2')
        const cellNames = _.map(currentDimension, 'cell_name')
        const trace = {
          x: X,
          y: Y,
          marker: {
            size: markerSize,
            symbol: 'circle',
            color: customColor
          },
          mode: 'markers',
          type: 'scatter',
          name: select + '_' + subcluster,
          text: cellNames
        }
        return trace
      }

      if (this.clusterCoordinatesSelect === 'All cell types') {
        const colormajorct = [
          '#3283FE',
          '#FBE426',
          '#782AB6',
          '#1C8356',
          '#16FF32',
          '#565656',
          '#a9a9a9',
          '#E2E2E2',
          '#C075A6',
          '#DEA0FD',
          '#85660D',
          '#FEAF16',
          '#1CBE4F',
          '#F6222E',
          '#90AD1C',
          '#325A9B',
          '#C4451C',
          '#FE00FA',
          '#222222'
        ]
        let mainct = _.map(this.ct, 'cell_type')
        mainct = unique(mainct)
        mainct = mainct.sort()
        const traceAll = []
        for (let i = 0; i < mainct.length; i++) {
          const tracetemp = getTrace(
            this.dimension,
            mainct[i],
            colormajorct[i],
            this.pointSize
          )
          traceAll.push(tracetemp)
        }
        // console.log(traceAll)
        return traceAll
      } else {
        const colorsubct = [
          '#E64B35B2',
          '#4DBBD5B2',
          '#00A087B2',
          '#3C5488B2',
          '#F39B7FB2',
          '#8491B4B2',
          '#91D1C2B2',
          '#DC0000B2',
          '#7E6148B2',
          '#0c66b2'
        ]
        // let mainct = _.map(this.ct, 'celltype')
        // mainct = unique(mainct)
        // mainct = mainct.sort() // how to define a group of subcluster

        // v1
        // let subct = _.map(this.ct, 'subcluster')

        // v2
        const subct = this.ct
          .filter((x) => x.cell_type === this.clusterCoordinatesSelect)
          .filter((x) => x.subcluster !== 'all')
          .map((x) => x.subcluster)
          .sort()
        // subct.shift()
        // console.log(subct)
        //  subct = unique(subct)
        //  subct.shift()

        const traceAll = []
        for (let i = 0; i < subct.length; i++) {
          const tracetemp = getSubclusterTrace(
            this.dimension,
            this.clusterCoordinatesSelect,
            subct[i],
            colorsubct[i],
            this.pointSize
          )
          traceAll.push(tracetemp)
        }
        return traceAll
      }
    },
    expressionDim() {
      const X = _.map(this.dimension, 'umap_1')
      const Y = _.map(this.dimension, 'umap_2')
      const cellNames = _.map(this.dimension, 'cell_name')
      /*
      // Actual gene expression object, this is much shorter than the dimension object
      const actualExpression = this.expression.reduce(
        (acc, cur) => ({ ...acc, [cur.cell_name]: cur.expression }),
        {}
      )
      // create a tmp object to set {cell_name:0}
      const tmpExpression = cellNames.reduce(
        (o, key) => Object.assign(o, { [key]: 0 }),
        {}
      )

      // Object.values(_.extend({}, tmpExpression, actualExpression))
*/
      // merge actualExpression and tmpExpression, overwrite 0 with actual expression values
      const trace = {
        x: X,
        y: Y,
        mode: 'markers',
        text: cellNames,
        marker: {
          size: this.pointSize,
          color: this.expression
        },
        colorscale: 'YlOrRd'
      }
      return [trace]
    }
  },
  methods: {
    async updateCluster() {
      const params = {
        id: this.$route.params.id,
        type: this.clusterCoordinatesSelect
      }
      await this.$store.dispatch('ad/fetchDimension', params)
    },
    async updateExpression() {
      await this.$store.dispatch('ad/fetchExpression', {
        gene: this.gene,
        id: this.dataset[0].data_id
      })
    }
  }
}
</script>

<style lang="scss" scoped></style>
