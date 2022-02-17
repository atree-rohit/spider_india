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
                <india-map-hex :map_data="data" :selected="selected" :popup="tooltip" :stateStats="stateStats" @stateSelected="selectState" />
            </v-col>
            <v-col cols="6">
                <taxa-sunburst :tree_data="data" :selected="selected" :popup="tooltip" @taxaSelected="selectTaxa" />
            </v-col>
        </v-row>
    </v-container>
</template>

<script>
import IndiaMapHex from './IndiaMapHex.vue'
import TaxaSunburst from './TaxaSunburst.vue'


import * as d3 from "d3";
import data from "../data/data.json";

export default {
    name: "AllSpecies",
    components: {
        IndiaMapHex,
        TaxaSunburst,
    },
    data() {
        return {
            data: data,
            tooltip: null,
            selected: {
                states:["All"],
                taxa:[],
                dates:[],
            }
        };
    },
    created() {
        console.clear()
        this.init()
        console.log(data[0])
    },
    computed: {
        taxonTree() {
            return d3.rollups(
                this.data,
                (o) => o.length,
                (o) => o.taxon_class_name,
                (o) => o.taxon_order_name,
                (o) => o.taxon_suborder_name,
                (o) => o.taxon_superfamily_name,
                (o) => o.taxon_family_name,
                (o) => o.taxon_subfamily_name,
                (o) => o.taxon_tribe_name,
                (o) => o.taxon_subtribe_name,
                (o) => o.taxon_genus_name,
                (o) => o.taxon_species_name
            );
        },
        stateStats() {
            let data = d3.groups(this.data, (o) => o.place_admin1_name)
            let op = []
            data.map((s) => {
                op[s[0]] = {
                    observations: s[1].length,
                    users: new Set(s[1].map((o) => o.user_id)).size,
                    species:new Set(s[1].map((o) => o.taxon_species_name)).size,
                }
            })
            op["All"] = {
                observations: 1,
                users: 1,
                species: 1,
            }
            return op
        }
    },
    methods:{
        init() {
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
        selectState(s) {
            this.selected.states = [s]
        },
        selectTaxa(t) {
            this.selected.taxa = t

        },
    }
};
</script>

