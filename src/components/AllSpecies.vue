<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
</style>

<template>
    <v-container fluid class="red lighten-4">
        {{selected}}
        <india-map :map_data="data" :selected_state="selected.states" :popup="tooltip" :stateStats="stateStats" @stateSelected="selectState" />
    </v-container>
</template>

<script>
import IndiaMap from './IndiaMap.vue'


import * as d3 from "d3";
import data from "../data/data.json";

export default {
    name: "AllSpecies",
    components: {
        IndiaMap
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
        console.log(this.stateStats);
        // console.log(this.data[0]);
        // console.log(this.taxonTree);
        this.init()
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
    }
};
</script>

