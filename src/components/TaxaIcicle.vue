<style scoped>
.breadcrumb {
  list-style: none;
  padding: 0;
  margin: 0;
}
.breadcrumb {
  background: #eee;
  border-width: 1px;
  border-style: solid;
  border-color: #f5f5f5 #e5e5e5 #ccc;
  border-radius: 5px;
  box-shadow: 0 0 2px rgba(0, 0, 0, 0.2);
  overflow: hidden;
  width: 100%;
}

.breadcrumb li {
  float: left;
}

.breadcrumb a {
  padding: 0.7em 1em 0.7em 2em;
  float: left;
  text-decoration: none;
  color: #303;
  position: relative;
  text-shadow: 0 1px 0 rgba(255, 255, 255, 0.5);
  background-color: #ddd;
  background-image: linear-gradient(to right, #7c9, #aec);
}

.breadcrumb li:first-child a {
  padding-left: 1em;
  border-radius: 5px 0 0 5px;
}

.breadcrumb a:hover {
  background: #fff;
}

.breadcrumb a::after,
.breadcrumb a::before {
  content: "";
  position: absolute;
  top: 50%;
  margin-top: -1.5em;
  border-top: 1.5em solid transparent;
  border-bottom: 1.5em solid transparent;
  border-left: 1em solid;
  right: -1em;
}

.breadcrumb a::after {
  z-index: 2;
  border-left-color: #aec;
}

.breadcrumb a::before {
  border-left-color: #5a3;
  right: -1.1em;
  z-index: 1;
}

.breadcrumb a:hover::after {
  border-left-color: #fff;
}

.breadcrumb .current,
.breadcrumb .current:hover {
  font-weight: bold;
  background: none;
}

.breadcrumb .current::after,
.breadcrumb .current::before {
  content: normal;
}

.breadcrumb li:hover a {
  color: #282 !important;
  text-decoration: none;
  text-shadow: 1px 1px 5px rgba(100, 255, 100, 0.25);
}
</style>
<template>
    <div class=''>
        <div id="icicleChart">
        </div>
        
    </div>
</template>

<script>
import * as d3 from "d3";
export default {
    name: "TaxaIcicle",
    props: ["data", "selected", "popup"],
    emits: ["taxaSelected"],
    data() {
        return {
            width: 0,
            height: 0,
            root: null,
            speciesData: null,
            mySVG: null,



            radius: 0,
            margin: {
                top: 50,
                right: 50,
                left: 50,
                bottom: 50,
            },
            g: null,
            path: null,
            parent: null,
            label: null,
            chartCode:null,
            breadcrumbs: [],
            watch_click: false,
            taxa_levels: ['Class', 'Order', 'Sub-Order', 'Super-Family', 'Family', 'Sub-Family', 'Tribe', 'Sub-Tribe', 'Genus', 'Species'],
            // taxa_levels: ["Super-family", "Family", "Sub-family", "Tribe", "Genus", "Species"],
        }
    },
    computed: {
        color() {
            // return d3.scaleOrdinal(d3.quantize(d3.interpolateRainbow, data.children.length + 1))
            return d3.scaleOrdinal(d3.quantize(d3.interpolateRainbow, this.speciesData.children.length +1))
        },
        color_old() {
            return d3.scaleOrdinal(
                d3.quantize(d3.interpolateWarm, this.speciesData.children.length + 4)
            )
        },
        tree_data() {
            let op = {}
            //d3.groups()
            this.data.map((o) => {
                if (op[o.taxon_species_name] === undefined) {
                    op[o.taxon_species_name] = o
                }
            })
            
            return Object.values(op)
        },
    },
    watch: {
    },
    mounted() {
        this.init()
        // this.chartCode = this.init().innerHTML
        
        // console.log(x, typeof(x))
                        

        // this.crumbClick("Papilionoidea")
    },
    methods: {
        init() {
            this.init_width_height()
            this.init_data()
            this.chart()
        },
        init_width_height() {
            /*
            this.width = window.innerWidth * 0.5
            this.height = window.innerHeight * 0.6
            if (window.innerWidth < 800) {
                this.width = window.innerWidth * 2
                this.height = window.innerHeight * 0.8
            }
            */
            this.width = 975
            this.height = 1200
        },
        init_data() {
            this.speciesData = this.initTree(this.data)
            this.root = this.partition(this.speciesData)
        },
        initTree(data) {
            let group = d3.groups(data,
                (d) => d.taxon_class_name,
                (d) => d.taxon_order_name,
                (d) => d.taxon_suborder_name,
                (d) => d.taxon_superfamily_name,
                (d) => d.taxon_family_name,
                (d) => d.taxon_subfamily_name,
                (d) => d.taxon_tribe_name,
                (d) => d.taxon_subtribe_name,
                (d) => d.taxon_genus_name,
                (d) => d.taxon_species_name)
            return this.hierarchy(group)
        },
        hierarchy(array) {
            let a = array[0]
            let op = { name: a[0] }            
            if(a[1][0].id === undefined){
                op["children"] = a[1].map((c) => this.hierarchy([c]))
            } else {
                op["size"] = a[1].length
            }
            return op
        },
        partition(data) {
            const root = d3.hierarchy(data)
                .sum(d => d.value)
                .sort((a, b) => b.height - a.height || b.value - a.value);  
            return d3.partition()
            .size([this.height, (root.height + 1) * this.width / 3])(root);

            // this.root = d3
            //     .hierarchy(data)
            //     .sum(() => 1)
            //     .sort((a, b) => b.value - a.value)
            // return d3.partition().size([2 * Math.PI, this.root.height + 1])(this.root)
        },
        chart() {
            const root = this.root;
            let focus = root;
            let width = this.width
            let height = this.height
            let format = d3.format(",d")
            let color = d3.scaleOrdinal(d3.quantize(d3.interpolateRainbow, this.speciesData.children.length + 1))

            // const svg = d3.create("svg")
            //     .attr("viewBox", [0, 0, width, height])
            //     .style("font", "10px sans-serif");
             if (!d3.select("#icicleChart svg").empty()) {
                    d3.selectAll("#icicleChart svg").remove()
                }
            const svg = d3.select("#icicleChart")
                .append("svg")
                .classed("bg-light text-center border boorder-primary rounded", true)
                // .attr("width", this.width)
                // .attr("height", this.height)
                .attr("viewBox", [0,0, this.width, this.height])

            const cell = svg
                .selectAll("g")
                .data(root.descendants())
                .join("g")
                .attr("transform", d => `translate(${d.y0},${d.x0})`);

            const rect = cell.append("rect")
                .attr("width", d => d.y1 - d.y0 - 1)
                .attr("height", d => rectHeight(d))
                .attr("fill-opacity", 0.6)
                .attr("fill", d => {
                    if (!d.depth) return "#ccc";
                    while (d.depth > 1) d = d.parent;
                    return color(d.data.name);
                })
                .style("cursor", "pointer")
                .on("click", (e,d) => clicked(e,d));

            const text = cell.append("text")
                .style("user-select", "none")
                .attr("pointer-events", "none")
                .attr("x", 4)
                .attr("y", 13)
                .attr("fill-opacity", d => +labelVisible(d));

            text.append("tspan")
                .text(d => d.data.name);

            const tspan = text.append("tspan")
                .attr("fill-opacity", d => labelVisible(d) * 0.7)
                .text(d => ` ${format(d.value)}`);

            cell.append("title")
                .text(d => `${d.ancestors().map(d => d.data.name).reverse().join("/")}\n${format(d.value)}`);

            function clicked(event, p) {
                let x = p
                focus = focus === p ? p = p.parent : p;
                
                root.each(d => d.target = {
                x0: (d.x0 - x.x0) / (x.x1 - x.x0) * height,
                x1: (d.x1 - x.x0) / (x.x1 - x.x0) * height,
                y0: d.y0 - x.y0,
                y1: d.y1 - x.y0
                });

                const t = cell.transition().duration(750)
                    .attr("transform", d => `translate(${d.target.y0},${d.target.x0})`);

                rect.transition(t).attr("height", d => rectHeight(d.target));
                text.transition(t).attr("fill-opacity", d => +labelVisible(d.target));
                tspan.transition(t).attr("fill-opacity", d => labelVisible(d.target) * 0.7);
            }
            
            function rectHeight(d) {
                return d.x1 - d.x0 - Math.min(1, (d.x1 - d.x0) / 2);
            }

            function labelVisible(d) {
                return d.y1 <= width && d.y0 >= 0 && d.x1 - d.x0 > 16;
            }
            
            console.log(svg.node())
            // return svg.node();
            },
        renderIcicle_1() {
            const width = this.width
            const height = this.height
            const root = this.root
            let focus  = root

            if (!d3.select("#icicleChart svg").empty()) {
                d3.selectAll("#icicleChart svg").remove()
            }
        
            
            
            // const svg = d3.create("svg")
            //     .attr("viewBox", [0, 0, width, height])
            //     .style("font", "10px sans-serif");

            const svg = d3.select("#icicleChart")
                    .append("svg")
                    // .classed("bg-light text-center border boorder-primary rounded", true)
                    // .attr("width", this.width)
                    // .attr("height", this.height)
                    .attr("viewBox", [0,0, this.width, this.height])
            
            const cell = svg
                .selectAll("g")
                .data(root.descendants())
                .join("g")
                .attr("transform", d => `translate(${d.y0},${d.x0})`);
                
            const rect = cell.append("rect")
            .attr("width", d => d.y1 - d.y0 - 1)
            .attr("height", d => rectHeight(d))
            .attr("fill-opacity", 0.6)
            .attr("fill", d => {
                console.log(this.color(d.data.name))
                if (!d.depth) return "#cc0";
                while (d.depth > 1) d = d.parent;
                return this.color(d.data.name);
                })
            .style("cursor", "pointer")
            .on("click", clicked);
            
            const text = cell.append("text")
                .style("user-select", "none")
                .attr("pointer-events", "none")
                .attr("x", 4)
                .attr("y", 13)
                .attr("fill-opacity", d => +labelVisible(d));
                
            text.append("tspan")
                .text(d => d.data.name);
                
            const tspan = text.append("tspan")
                .attr("fill-opacity", d => labelVisible(d) * 0.7)
                .text(d => ` ${format(d.value)}`);
                
            cell.append("title")
                .text(d => `${d.ancestors().map(d => d.data.name).reverse().join("/")}\n${format(d.value)}`);
                
            function clicked(event, p) {
                focus = focus === p ? p = p.parent : p;
                console.log(event, p)
                root.each(d => d.target = {
                    x0: (d.x0 - p.x0) / (p.x1 - p.x0) * height,
                    x1: (d.x1 - p.x0) / (p.x1 - p.x0) * height,
                    y0: d.y0 - p.y0,
                    y1: d.y1 - p.y0
                });
                
                const t = cell.transition().duration(750)
                    .attr("transform", d => `translate(${d.target.y0},${d.target.x0})`);
                rect.transition(t).attr("height", d => rectHeight(d.target));
                text.transition(t).attr("fill-opacity", d => +labelVisible(d.target));
                tspan.transition(t).attr("fill-opacity", d => labelVisible(d.target) * 0.7);
            }

            function rectHeight(d) {
                return d.x1 - d.x0 - Math.min(1, (d.x1 - d.x0) / 2);
            }
            
            function labelVisible(d) {
                return d.y1 <= width && d.y0 >= 0 && d.x1 - d.x0 > 16;
            }

            function format(d){
                return d3.format(`, ${d}`)
            }
            
            console.log(svg.node())                        
        },
    },
}
</script>