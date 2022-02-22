<style>
    #map-container{
        background-color: rgb(190, 229, 235);
        transition: all 2.5s;
    }
    #map-container .map-states path{
        stroke-width:0.25px;
        stroke:black;
    }
    #map-container .map-states path:hover,
    #map-container .map-hex path:hover
     {
        cursor: pointer;
        fill: #fff;
        stroke: black;
    }
    #map-container .map-states path.state-selected {
        fill: #59f !important;
        stroke: rgba(205,105,0,1);
        stroke-width:0.1px;
    }

    #map-container .map-text text,
    .d3-tooltip {
        font-family: Arial, Helvetica, sans-serif;
    }

    #map-container .map-text text{
        fill: #545;
        font-size: 1vw;
        transition: fill,text-shadow .125s;
    }

    #map-container .map-text text:hover {
        fill: #00c;
        cursor: pointer;
    }

    #map-container .map-hex{
        opacity: 0.75;
    }
    #map-container .map-hex path{
        /* stroke: black; */
        stroke-width: 0.5px;
    }
    #map-container .map-hex_text text{
        font-size: 0.65vw;
        fill :white;
        text-shadow: 0 0 2px black;
    }
    #map-container .map-points circle{
        stroke: black;
        stroke-width: 0.125px;
        fill:  rgba(255,0,0,.5);
    }
    .d3-tooltip table, 
    .d3-tooltip td{
        border: 0.5px solid #666;
        border-collapse: collapse;
        padding:2.5px;
    }
    .d3-tooltip table td ol{
        list-style-position: inside;
    }
    @media screen and (max-width: 800px) {
        #map-container .map-text{
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
import {geoToH3, h3ToGeoBoundary, h3ToGeo} from 'h3-js';
import country from '../data/country.json'

export default {
    name: "IndiaMapHex",
    props: [
        "map_data",
        "selected",
        "popup",
    ],
    data() {
        return {
            svg_layers:{
                states: null,
                text: null,
                hex: null,
                hex_text: null,
            },
            svg: null,
            height: 0,
            width: 0,
            h3_zoom: 3,
            h3_hexagons: [],
            tooltip:null,
        }
    },
    emits: ["stateSelected"],
    mounted() {
        this.init()

        //cleaning places
        /*
        let place_counts = {
            raw:{},
            rounded: {},
        }
        for (const num of this.map_data.map((o) => `${o.latitude}-${o.longitude}`)) {
            place_counts.raw[num] = place_counts.raw[num] ? place_counts.raw[num] + 1 : 1;
        }
        for (const num of this.map_data.map((o) => `${parseFloat(o.latitude).toFixed(3)}-${parseFloat(o.longitude).toFixed(3)}`)) {
            place_counts.rounded[num] = place_counts.rounded[num] ? place_counts.rounded[num] + 1 : 1;
        }

        console.log(place_counts)
        console.log(this.map_data.length, Object.keys(place_counts.raw).length, Object.keys(place_counts.rounded).length)
        */
    },
    computed: {
        stateStats() {
            let data = d3.groups(this.filteredData, (o) => o.place_admin1_name)
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
        selected_state() {
            return (this.selected.states.length == 0 || this.selected.states[0] == "All") ? "All" : this.selected.states[0]
        },
        selectedGeoJson() {
            return (this.selected_state === "All")
                    ? {properties:{ST_NM: 'All'}}
                    : Object.values(country.features).filter((s) => this.stateName(s) === this.selected_state)[0]
        },
        zoom() {
            let svg = this.svg
            return d3.zoom()
                    .scaleExtent([.5, 50])
                    .translateExtent([[-0.5 * this.width,-0.5 * this.height],[2.5 * this.width, 2.5 * this.height]])
                    .on('zoom', (event) => {
                        svg.selectAll('.map-text')
                            .attr('transform', event.transform),
                        svg.selectAll('.map-hex_text')
                            .attr('transform', event.transform),
                        svg.selectAll('path')
                            .attr('transform', event.transform),
                        svg.selectAll('circle')
                            .attr('transform', event.transform)
                            .attr("r", 4 / event.transform.k)
                    })            
        },
        filteredData(){
            let op = this.map_data
            // let taxa_levels = ['superfamily', 'family', 'subfamily', 'tribe', 'subtribe', 'genus', 'species']
            // let taxon_levels = ['class', 'order', 'suborder', 'superfamily', 'family', 'subfamily', 'tribe', 'subtribe', 'genus', 'species']
            let taxon_levels = ['order', 'superfamily', 'family', 'subfamily', 'tribe', 'genus', 'species']
            this.selected.taxa.map((v,k) => {
                op = op.filter((o) => o[`taxon_${taxon_levels[k]}_name`] === v)
            })
            if(this.selected.states != 'All'){
                op = op.filter((o) => this.selected.states.indexOf(o.place_admin1_name) !== -1)
            }
            
            return op
        },
        stateData() {
            return d3.groups(this.filteredData, (o) =>  o.place_admin1_name)
        },
        colors() {
            let state_max = d3.max(this.stateData.map((s) => s[1].length))
            // return d3.scaleLinear()
            //     .domain([0, 1, state_max * 0.25, state_max])
            //     .range(["#f77", "#ca0", "#ada", "#3d3"])
            return d3.scaleLog()
                .domain([0, 1, state_max])
                // .range(["#f55","#f50", "#5f5"])
                .range(["#000","#555", "#ccc"])
                // .interpolator(d3.interpolateGreens);
        },
        hexColors() {
            let hex = {
                max: d3.max(this.h3_hexagons.map((h) => h.rows.length)),
                min: d3.min(this.h3_hexagons.map((h) => h.rows.length))
            }
            return d3.scaleLog()
                .domain([hex.min, hex.max])
                .range(["#f00", "#0f0"])
            // return d3.scaleSequential()
            //     .domain([-hex.max/2, hex.max])
            //     .interpolator(d3.interpolateGreys)
                
        },
        projection(){
            return d3.geoMercator().scale(850).center([87, 25.5])
        },
        path() {
            return d3.geoPath().projection(this.projection)
        },
    },
    watch: {
        map_data() {
            this.init()
        },
        'selected.states': function (){
            this.mapPoints()
            this.mapHexPoints()
        },
        'selected.taxa': function (){
            this.init()
            // this.mapPoints()
            // this.mapHexPoints()
        },
    },
    methods:{
        init() {
            this.height = window.innerHeight * 0.85
            this.width = window.innerWidth * 0.475
            this.tooltip = this.popup

            if(window.innerWidth < 800){
                this.height = window.innerHeight * 0.5
                this.width = window.innerWidth * 0.9
            }

            this.renderMap()
            this.initLegend()
            this.clicked(this.selectedGeoJson)
            this.svg.call(this.zoom)
            this.mapHexPoints()
            this.mapPoints()
        },
        initLegend() {
            
            let op = d3Legend.legendColor()
                                .shapeWidth(45)
                                .scale(this.colors)
                                .labelFormat(d3.format(".0f"))
                                .orient('horizontal')
                                .labelOffset(-10)
                                .labelAlign("center")
                                .cells(6)
                                // .shapePadding(47)
            if(this.height > this.width) {
                op.shapeWidth(35)
                .cells(4)
            }


            this.svg.append("g").classed("map-legend-1", true)
                .attr("transform", `translate(${this.width * 0.5}, 50)`)
                .call(op)
        },
        renderMap() {
            if (!d3.select("#map-container svg").empty()) {
                d3.selectAll("#map-container svg").remove()
            }
            this.svg = d3.select("#map-container")
                        .append("svg")
                        .attr("preserveAspectRatio", "xMinYMin meet")
                        .attr("width", this.width)
                        .attr("height", this.height)
                        .classed("svg-content d-flex m-auto", true)

            this.svg_layers.states = this.svg.append("g").classed("map-states", true).selectAll("path").append("g")
            this.svg_layers.text = this.svg.append("g").classed("map-text", true).selectAll("text").append("g")
            this.svg_layers.hex = this.svg.append("g").classed("map-hex", true).selectAll("path").append("g")
            this.svg_layers.hex_text = this.svg.append("g").classed("map-hex_text", true).selectAll("text").append("g")
            
            country.features.map((state) => {
                if(this.selected_state === "All") {
                    this.drawPolygonText(state)
                }
                this.drawPolygon(state)
            })
        },
        mouseActions(root, state_name){
            root.on('mouseover', () => this.tooltip.html(this.tooltipText(state_name)).style('visibility', 'visible'))
                    .on('mousemove', (event) => {
                        this.tooltip
                        .style('top', event.pageY - 10 + 'px')
                        .style('left', event.pageX + 10 + 'px');
                    })
                    .on('mouseout', () => this.tooltip.html(``).style('visibility', 'hidden'))
                    .on("click", (e,d) => this.clicked(d))
        },
        drawPolygon(state){
            let name = this.stateName(state)
            let op = this.svg_layers.states.append("g")
                    .data([state])
                    .enter().append("path")
                    .attr("d", this.path)
                    .attr("id", this.stateID(name))
            this.mouseActions(op, name)
                
            {(name == this.selected_state) ? op.classed("state-selected", true) : op.attr("fill", () => this.colors(this.noOfObservations(name)))}
        },
        drawPolygonText(state){
            let name = this.stateName(state)
            let op = this.svg_layers.text.append("g")
                        .data([state])
                        .enter().append("text")
                        .attr("x", (h) => this.path.centroid(h)[0] )
                        .attr("y", (h) => this.path.centroid(h)[1] )
                        .attr("text-anchor", "middle")
                        .text(this.noOfObservations(name))
            this.mouseActions(op, name)
        },
        noOfObservations(state_name){
            let state_observations = this.stateData.filter((s) => s[0] === state_name)[0]
            return (state_observations === undefined) ? 0 : state_observations[1].length
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
        stateName(state){
            // console.log(state)
            return state.properties.ST_NM
        },
        clicked(d) {
            this.tooltip.html(``).style('visibility', 'hidden')
            let state = this.stateName(d)
            let [[x0, y0], [x1, y1]] = [[0,0],[0,0]]

            this.svg_layers.states.transition().style("fill", null)

            if(d3.select(".state-selected")["_groups"][0][0] != null){
                d3.select("#" + d3.select(".state-selected")["_groups"][0][0].id).attr("class", null)
            }

            if(this.selected_state == state){
                [[x0, y0], [x1, y1]] = this.path.bounds(country)
            } else {
                [[x0, y0], [x1, y1]] = this.path.bounds(d)
                d3.select("#" + this.stateID(state)).classed("state-selected", true)
            }
            if(this.selected_state == state){
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
        mapHexPoints() {
            if (!d3.select("#map-container svg g.map-hex").empty()) {
                d3.selectAll("#map-container svg g.map-hex").remove()
                d3.selectAll("#map-container svg g.map-hex_text").remove()
            }
            this.h3_hexagons = []

            this.svg_layers.hex = this.svg.append("g").classed("map-hex", true).selectAll("path").append("g")
            this.svg_layers.hex_text = this.svg.append("g").classed("map-hex_text", true).selectAll("text").append("g")

            this.filteredData.map((p) => {
                const h3Address = geoToH3(p.latitude, p.longitude, this.h3_zoom)
                let matchID = this.h3_hexagons.findIndex((h) => h.hexID === h3Address)
                let row = null
                
                
                if(p.taxa_name != null || p.taxa_name != ""){
                    row = this.parseData(p)
                }

                if (matchID == -1) {
                    let h3Geo = this.hexFeatures(h3ToGeoBoundary(h3Address, true))
                    this.h3_hexagons.push({ "hexID": h3Address, "coordinates": h3Geo, "rows": [row] })
                }
                else {
                    this.h3_hexagons[matchID].rows.push(row)
                }
                        
            })
            this.h3_hexagons.map((h) => {
                this.drawHex(h)
                // if(this.h3_zoom < 4){
                //     this.drawHexText(h)
                // }
            })
                
        },
        drawHex(h){
            let op = this.svg_layers.hex.append("g")
                    .data([h.coordinates])
                    .enter().append("path")
                    .attr("fill", () => this.hexColors(h.rows.length).replace('rgb', 'rgba').replace(')', ', 0.75)'))
                    // .attr("stroke", () => this.hexColors(h.rows.length))
                    .attr("d", this.path)
            this.hexMouseActions(op,h)
        },
        drawHexText(h){
            let op = this.svg_layers.hex_text.append("g")
                .data([h.coordinates])
                .enter().append("text")
                .attr("x", (h) => this.path.centroid(h)[0] )
                .attr("y", (h) => this.path.centroid(h)[1] )
                .attr("text-anchor", "middle")
                .text(h.rows.length)
            this.hexMouseActions(op,h)
        },
        hexMouseActions(root,h){
            root.on('mouseover', () => this.tooltip.html(this.tooltipHexText(h)).style('visibility', 'visible'))
                    .on('mousemove', (event) => {
                        this.tooltip
                        .style('top', event.pageY - 10 + 'px')
                        .style('left', event.pageX + 10 + 'px');
                    })
                    .on('mouseout', () => this.tooltip.html(``).style('visibility', 'hidden'))
                    // .on("click", (e,d) => this.clicked(d))
        },
        tooltipHexText(h){
            let op = [
                ["Observations", h.rows.length],
                ["Unique Species", Array.from(new Set(h.rows.map(r => r.scientific_name))).length],
                ["Users", Array.from(new Set(h.rows.map(r => r.user))).length],
                ["Place Names", "<ol><li>" + Array.from(new Set(h.rows.map(r => r.place))).join("</li><li>") + "</li></ol>"]
            ]
            return `<table>${op.map((r)=> '<tr><td>'+r.join('</td><td>') + '</td></tr>').join('')}</table>`

        },
        parseData(data){
            return {
                "id": data.id,
                "scientific_name": data.taxon_species_name,
                "common_name": data.common_name,
                "image": data.img_url,
                "user": data.user_login,
                "place": data.place_guess
                // "uri": `https://www.inaturalist.org/observations/${data.id}`
            }
        },
        hexFeatures(array) {
            const geoJSON = {
                "type": "FeatureCollection",
                "features": [{
                    "type": "Feature",
                    "geometry": {
                        "type": "Polygon",
                        "coordinates": [array.reverse()]
                    }
                }]
            }
            return geoJSON;
        },
        getLatLon(observation){
            return(h3ToGeo(observation.hexID));
        },
        mapPoints(){

            let points = []

            if (!d3.select("#map-container .map-points").empty()) {
                d3.selectAll(".map-points").remove()
            }
            
            if(this.selected_state != 'All'){
                this.stateData.filter((s) => s[0] === this.selected_state)[0][1].map((o) => {
                    // points.push(o.latitude, o.longitude, o.id, o.place_guess)
                    points.push([o.longitude, o.latitude, o.id, o.place_guess])
                })
            } else {
                this.stateData.map((s) => {
                    s[1].map((o) => points.push([o.longitude, o.latitude, o.id, o.place_guess]))
                })
                // points = 
                // console.log("SD",this.stateData)
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