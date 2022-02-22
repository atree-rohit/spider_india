<style>
    #map-container .state-selected {
        /*fill: #afa;*/
        fill: #fff;
        stroke: rgba(205,105,0,1);
        stroke-width:1.5px;
    }
    #map-container .poly_text {
        fill: #545;
        font-size: 0.5vw;
        transition: fill,text-shadow .125s;
    }
    #map-container .poly_text:hover {
        fill: #00c;
        cursor: pointer;
    }
    #map-container .map-boundary{
        transition: all .25s;
    }
    #map-container .map-boundary path:hover {
        cursor: pointer;
        fill: rgba(250,50,50,1);
    }
    #map-container .map-points circle{
        stroke: red;
        stroke-width: 0.125px;
        fill:  rgba(255,0,0,.5);
    }
    @media screen and (max-width: 800px) {
        #map-container .poly_text{
            font-size: 3.5vw;
        }
    }
</style>

<template>
    <div>
        <div id="map-container" />
    </div>
</template>

<script>
import * as d3 from 'd3'
import * as d3Legend from "d3-svg-legend"
import country from '../data/country.json'

export default {
    name: "IndiaMap",
    props: [
        "map_data",
        "selected_state",
        "popup",
    ],
    data() {
        return {
            states: null,
            path: null,
            svg: {},
            projection: {},
            colors: {},
            height: 0,
            width: 0,
            tooltip:null,
        }
    },
    emits: ["stateSelected"],
    mounted() {
        console.clear();
        this.init()
        // console.log(`#map-container dimentions : ${this.width} x ${this.height}`)
    },
    computed: {
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
        },
        selected() {
            return (this.selected_state.length == 0 || this.selected_state[0] == "All") ? "All" : this.selected_state[0]
        },
        selectedGeoJson() {
            let state_key = (this.selected !== 'All') ? Object.values(country.features).filter((c) => c.properties.ST_NM === this.selected) : null
            let op = (state_key === null) ? {properties:{ST_NM: 'All'}} : country.features[state_key]            
            return op
        },
        zoom() {
            let svg = this.svg
            return d3.zoom()
                    .scaleExtent([.5, 50])
                    .translateExtent([[-0.5 * this.width,-0.5 * this.height],[2.5 * this.width, 2.5 * this.height]])
                    .on('zoom', (event) => {
                        svg.selectAll('.poly_text')
                            .attr('transform', event.transform),
                        svg.selectAll('path')
                            .attr('transform', event.transform),
                        svg.selectAll('circle')
                            .attr('transform', event.transform)
                            .attr("r", 4 / event.transform.k)
                    })            
        },
        stateData() {
            return d3.groups(this.map_data, (o) =>  o.place_admin1_name)
                
        },
    },
    watch: {
        map_data() {
            this.init()
        },
        selected_state() {
            this.mapPoints()
        }
    },
    methods:{
        init() {
            let state_max = 0

            this.height = window.innerHeight * 0.85
            this.width = window.innerWidth * 0.475
            this.tooltip = this.popup

            if(window.innerWidth < 800){
                this.height = window.innerHeight * 0.5
                this.width = window.innerWidth * 0.9
            }
            
            this.stateData.map((s) => {
                if(s[1].length > state_max){
                    state_max = s[1].length
                }
            })

            this.colors = d3.scaleLinear()
                .domain([0, 1, state_max*.25, state_max])
                .range(["#f77", "#ca0", "#ada", "#3d3"])

            this.renderMap()
            this.clicked(this.selectedGeoJson)
        },
        initLegend() {
            let op = d3Legend.legendColor()
                                .shapeWidth(45)
                                .scale(this.colors)
                                .labelFormat(d3.format(".0f"))
                                .orient('horizontal')
                                .labelOffset(-10)
                                .labelAlign("start")
                                .cells(6)
                                // .shapePadding(47)
            if(this.height > this.width) {
                op.shapeWidth(35)
                .cells(4)
            }
            return op
        },
        renderMap() {
            if (!d3.select("#map-container svg").empty()) {
                d3.selectAll("#map-container svg").remove()
            }
            let tooltip = this.tooltip

            this.svg = d3.select("#map-container")
                        .append("svg")
                            .attr("preserveAspectRatio", "xMinYMin meet")
                            .attr("width", this.width)
                            .attr("height", this.height)
                            .style("background-color", "rgb(190, 229, 235)")
                            .classed("svg-content d-flex m-auto", true)

            this.projection = d3.geoMercator().scale(850).center([87, 25.5])
            this.path = d3.geoPath().projection(this.projection)

            let base = this.svg.append("g")
                .classed("map-boundary", true)

            let base_text = base.selectAll("text").append("g")
            base = base.selectAll("path").append("g")
            this.states = base.append("g").classed("states", true)
            
            country.features.map((state) => {
                let s_name = state.properties.ST_NM
                let state_observations = this.stateData.filter((s) => s[0] === s_name)[0]
                let no_of_observations = (state_observations === undefined) ? 0 : state_observations[1].length
                let current_state = this.states.append("g")
                    .data([state])
                    .enter().append("path")
                    .attr("d", this.path)
                    .attr("id", this.stateID(s_name))
                    .attr("title", s_name)
                    .on("click", (e,d) => this.clicked(d))
                    .on('mouseover', () => tooltip.html(this.tooltipText(s_name)).style('visibility', 'visible'))
                    .on('mousemove', (event) => {
                        tooltip
                        .style('top', event.pageY - 10 + 'px')
                        .style('left', event.pageX + 10 + 'px');
                    })
                    .on('mouseout', () => tooltip.html(``).style('visibility', 'hidden'))
                
                if(this.selected === "All") {
                    base_text.append("g")
                        .data([state])
                        .enter().append("text")
                        .classed("poly_text", true)
                        .attr("x", (h) => this.path.centroid(h)[0] )
                        .attr("y", (h) => this.path.centroid(h)[1] )
                        .attr("text-anchor", "middle")
                        .text(no_of_observations)
                        .on('mouseover', () => tooltip.html(this.tooltipText(s_name)).style('visibility', 'visible'))
                        .on('mousemove', (event) => {
                            tooltip
                            .style('top', event.pageY - 10 + 'px')
                            .style('left', event.pageX + 10 + 'px');
                        })
                        .on('mouseout', () => tooltip.html(``).style('visibility', 'hidden'))
                        .on("click", (e,d) => this.clicked(d))
                }

                if (s_name == this.selected) {
                    current_state.classed("state-selected", true)
                } else {
                    current_state.attr("fill", () => this.colors(no_of_observations))
                }
            })
            let legend = this.initLegend()
            this.svg.append("g")
                .attr("transform", "translate("+this.width*.5+", 50)")
                .call(legend)
            this.svg.call(this.zoom)
        },
        tooltipText(state){
            let op = "NO DATA!!"
            if(this.stateStats[state] !== undefined){
                op = `<table>
            <tr><td><strong>State</strong></td><td class='text-center'>${state}</td></tr>
            <tr><td><strong>Observations</strong></td><td class='text-center'>${this.stateStats[state].observations}</td></tr>
            <tr><td><strong>Users</strong></td><td class='text-center'>${this.stateStats[state].users}</td></tr>
            <tr><td><strong>Unique</strong> Taxa</td><td class='text-center'>${this.stateStats[state].species}</td></tr>
            </table>`
            }
            return op
        },
        stateID(s){
            return s.replaceAll(" ", "_").replaceAll("&", "")
        },
        clicked(d) {
            this.tooltip.html(``).style('visibility', 'hidden')
            let state = d.properties.ST_NM
            let [[x0, y0], [x1, y1]] = [[0,0],[0,0]]

            this.states.transition().style("fill", null)

            if(d3.select(".state-selected")["_groups"][0][0] != null){
                d3.select("#" + d3.select(".state-selected")["_groups"][0][0].id).attr("class", null)
            }

            if(this.selected == state){
                [[x0, y0], [x1, y1]] = this.path.bounds(country)
            } else {
                [[x0, y0], [x1, y1]] = this.path.bounds(d)
                d3.select("#" + this.stateID(state)).classed("state-selected", true)
            }
            if(this.selected == state){
                this.$emit('stateSelected', 'All')
            } else {
                this.$emit('stateSelected', state)
            }
            
            this.svg.transition().duration(750).call(
                this.zoom.transform,
                d3.zoomIdentity
                .translate(this.width / 2, this.height / 2)
                .scale(Math.min(8, 0.9 / Math.max((x1 - x0) / this.width, (y1 - y0) / this.height)))
                .translate(-(x0 + x1) / 2, -(y0 + y1) / 2),
            )
        },
        mapPoints(){
            let points = []

            if (!d3.select("#map-container .map-points").empty()) {
                d3.selectAll(".map-points").remove()
            }
            
            if(this.selected != 'All'){
                this.stateData.filter((s) => s[0] === this.selected)[0][1].map((o) => {
                    // points.push(o.latitude, o.longitude, o.id, o.place_guess)
                    points.push([o.longitude, o.latitude, o.id, o.place_guess])
                })
                
            }
            
            if(points.length > 0){
                this.svg.append('g')
                    .classed('map-points', true)
                    .selectAll("circle")
                    .data(points).enter()
                    .append("circle")
                    .attr("cx", (d) => this.projection(d)[0])
                    .attr("cy", (d) => this.projection(d)[1])
                    .attr("r", "3px")
                    .on("click", (e,d) => this.gotoObservation(d))
            }
        },
        gotoObservation(d){
            window.open(`https://www.inaturalist.org/observations/${d[2]}`, '_blank');
        }
    }
};
</script>