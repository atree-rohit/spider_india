<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
</style>

<template>
    <v-container fluid class="red lighten-4">
        <v-row>
            <v-col cols="12">
                {{selected}}
            </v-col>
        </v-row>
        <v-row>
            <v-col cols="6">
                <india-map-hex :map_data="data" :selected="selected" :popup="tooltip" @stateSelected="selectState" />
            </v-col>
            <v-col cols="6">
                <v-row>
                    <v-col cols="12" class="text-center pa-0">
                        <v-btn-toggle
                            v-model='taxa_chart_type'
                             active-class='green'
                        >
                            <v-btn v-for='t in taxa_chart_types' :key="t" v-text="`${t} Chart`" :value="t" />
                        </v-btn-toggle>
                    </v-col>
                    <v-col cols="12" class="pa-0">
                        <taxa-sunburst :data="data" :selected="selected" :popup="tooltip" @taxaSelected="selectTaxa" v-if="taxa_chart_type =='Sunburst'" />
                        <taxa-icicle :data="data" :selected="selected" :popup="tooltip" @taxaSelected="selectTaxa" v-else-if="taxa_chart_type =='Icicle'" />
                        <tree-of-life :data="data" :selected="selected" :popup="tooltip" @taxaSelected="selectTaxa" v-else-if="taxa_chart_type =='TreeOfLife'" />
                    </v-col>
                </v-row>
            </v-col>
        </v-row>
    </v-container>
</template>

<script>
import IndiaMapHex from './IndiaMapHex.vue'
import TaxaSunburst from './TaxaSunburst.vue'
import TaxaIcicle from './TaxaIcicle.vue'
import TreeOfLife from './TreeOfLife.vue'


import * as d3 from "d3";
import raw_data from "../data/data.json";

export default {
    name: "AllSpecies",
    components: {
        IndiaMapHex,
        TaxaSunburst,
        TaxaIcicle,
        TreeOfLife,
    },
    data() {
        return {
            data: [],
            tooltip: null,
            selected: {
                states:["All"],
                taxa:["Araneae"],
                dates:[],
            },
            taxa_chart_type:"",
            taxa_chart_types:["Sunburst", "Icicle", "TreeOfLife"],

        };
    },
    created() {
        console.clear()
        this.init()
        // console.log(data[0])
    },
    methods:{
        init() {
            this.cleanStateNames()
            this.tooltip = d3.select('body')
                .append('div')
                .attr('class', 'd3-tooltip')
                .style('position', 'absolute')
                .style('z-index', '10')
                .style('visibility', 'hidden')
                .style('padding', '10px')
                .style('background', 'rgba(0,0,0,0.6)')
                .style('border-radius', '4px')
                .style('color', '#fff')
                .text('a simple tooltip')
        },
        cleanStateNames() {
            let fix_names = {
                'Dadra and Nagar Haveli': 'Dadra and Nagar Haveli and Daman and Diu',
                'NCT of Delhi': 'Delhi'
            }
            this.data = raw_data.map((o) => {
                if(Object.keys(fix_names).indexOf(o.place_admin1_name) !== -1){
                    o.place_admin1_name = fix_names[o.place_admin1_name]                    
                }
                return o
            })
        },
        selectState(s) {
            this.selected.states = [s]
        },
        selectTaxa(t) {
            this.selected.taxa = t

        },
    }
};
</script>

