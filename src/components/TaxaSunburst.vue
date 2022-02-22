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
    <div>
        <div id="sunburstChart">
        </div>
        
    </div>
</template>

<script>
import * as d3 from "d3";
export default {
    name: "TaxaSunburst",
    props: ["data", "selected", "popup"],
    emits: ["taxaSelected"],
    data() {
        return {
            width: 0,
            height: 0,
            radius: 0,
            margin: {
                top: 50,
                right: 50,
                left: 50,
                bottom: 50,
            },

            root: null,
            g: null,
            path: null,
            parent: null,
            label: null,

            chartCode:null,
            speciesData: [],
            breadcrumbs: [],
            watch_click: false,
            taxa_levels: ['Class', 'Order', 'Sub-Order', 'Super-Family', 'Family', 'Sub-Family', 'Tribe', 'Sub-Tribe', 'Genus', 'Species'],
            // taxa_levels: ["Super-family", "Family", "Sub-family", "Tribe", "Genus", "Species"],
        }
    },
    computed: {
        arc() {
            return d3
                .arc()
                .startAngle((d) => d.x0)
                .endAngle((d) => d.x1)
                .padAngle((d) => Math.min((d.x1 - d.x0) / 2, 0.01))
                .padRadius(this.radius * 1.5)
                .innerRadius((d) => d.y0 * this.radius)
                .outerRadius((d) => Math.max(d.y0 * this.radius, d.y1 * this.radius))
        },
        color() {
            console.log("colors", this.speciesData.children.length + 4)
            return d3.scaleOrdinal(
                d3.quantize(d3.interpolateWarm, this.speciesData.children.length + 1)
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
        selected(newVal) {
            if (newVal.length == 0) {
                this.crumbClick("Papilionoidea")
            }
        },
        tree_data() {
            this.init()
            
            this.watch_click = true
            this.crumbClick(this.selected[this.selected.length - 1])
            this.watch_click = false
        },
    },
    mounted() {
        this.init()
    },
    methods: {
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
        hierarchy(array){
            let a = array[0]
            let op = { name: a[0] }            
            if(a[1][0].id === undefined){
                op["children"] = a[1].map((c) => this.hierarchy([c]))
            } else {
                op["size"] = a[1].length
            }
            return op
        },
        init(){
            this.width = window.innerWidth * 0.5
            this.height = window.innerHeight * 0.6
            if (window.innerWidth < 800) {
                this.width = window.innerWidth * 2
                this.height = window.innerHeight * 0.8
            }

            this.speciesData = this.initTree(this.data)
            this.root = this.partition(this.speciesData);
            this.radius = d3.min([this.width, this.height])/6
            this.root.each(d => d.current = d);
            
            // const arc = d3
            //     .arc()
            //     .startAngle((d) => d.x0)
            //     .endAngle((d) => d.x1)
            //     .padAngle((d) => Math.min((d.x1 - d.x0) / 2, 0.01))
            //     .padRadius(this.radius * 1.5)
            //     .innerRadius((d) => d.y0 * this.radius)
            //     .outerRadius((d) => Math.max(d.y0 * this.radius, d.y1 * this.radius))

             if (!d3.select("#sunburstChart svg").empty()) {
                d3.selectAll("#sunburstChart svg").remove()
            }
            const svg = d3.select("#sunburstChart")
                        .append("svg")
                        .attr("preserveAspectRatio", "xMinYMin meet")
                        .attr("width", this.width)
                        .attr("height", this.height)
                        .classed("svg-content d-flex m-auto", true)
            
            // const svg = d3.create("svg")
            // .attr("viewBox", [0, 0, this.width, this.height])
            // .style("font", "10px sans-serif");

            this.g = svg.append("g")
                .attr("transform", `translate(${this.width / 2},${this.height / 2})`);

            this.path = this.g.append("g")
            .selectAll("path")
            .data(this.root.descendants().slice(1))
            .join("path")
            .attr("fill", d => { while (d.depth > 1) d = d.parent; return this.color(d.data.name); })
            .attr("fill-opacity", d => this.arcVisible(d.current) ? (d.children ? 0.6 : 0.4) : 0)
            .attr("pointer-events", d => this.arcVisible(d.current) ? "auto" : "none")

            .attr("d", d => this.arc(d.current));

            this.path.filter(d => d.children)
            .style("cursor", "pointer")
            .on("click", this.clicked);

            this.path.append("title")
            .text(d => `${d.ancestors().map(d => d.data.name).reverse().join("/")}\n${this.format(d.value)}`);

            this.label = this.g.append("g")
                .attr("pointer-events", "none")
                .attr("text-anchor", "middle")
                .style("user-select", "none")
                .selectAll("text")
                .data(this.root.descendants().slice(1))
                .join("text")
                .attr("dy", "0.35em")
                .attr("fill-opacity", d => +this.labelVisible(d.current))
                .attr("transform", d => this.labelTransform(d.current))
                .text(d => d.data.name);

            this.parent = this.g.append("circle")
            .datum(this.root)
            .attr("r", this.radius)
            .attr("fill", "none")
            .attr("pointer-events", "all")
            .on("click", this.clicked);

        },
        clicked(event, p) {
            this.parent.datum(p.parent || this.root);

            this.root.each(d => d.target = {
                x0: Math.max(0, Math.min(1, (d.x0 - p.x0) / (p.x1 - p.x0))) * 2 * Math.PI,
                x1: Math.max(0, Math.min(1, (d.x1 - p.x0) / (p.x1 - p.x0))) * 2 * Math.PI,
                y0: Math.max(0, d.y0 - p.depth),
                y1: Math.max(0, d.y1 - p.depth)
            });

            const t = this.g.transition().duration(750);
            let that = this

            // Transition the data on all arcs, even the ones that aren’t visible,
            // so that if this transition is interrupted, entering arcs will start
            // the next transition from the desired position.
            this.path.transition(t)
                .tween("data", d => {
                    const i = d3.interpolate(d.current, d.target);
                    return t => d.current = i(t);
                })            
                .filter(function(d) {
                    return +this.getAttribute("fill-opacity") || that.arcVisible(d.target);
                })
                .attr("fill-opacity", d => that.arcVisible(d.target) ? (d.children ? 0.6 : 0.4) : 0)
                .attr("pointer-events", d => that.arcVisible(d.target) ? "auto" : "none") 
                .attrTween("d", d => () => that.arc(d.current));

            this.label.filter(function(d) {
                return +this.getAttribute("fill-opacity") || that.labelVisible(d.target);
            }).transition(t)
                .attr("fill-opacity", d => +that.labelVisible(d.target))
                .attrTween("transform", d => () => that.labelTransform(d.current));
        },
        arcVisible(d) {
            return d.y1 <= 3 && d.y0 >= 1 && d.x1 > d.x0;
        },
        labelVisible(d) {
            return d.y1 <= 3 && d.y0 >= 1 && (d.y1 - d.y0) * (d.x1 - d.x0) > 0.03;
        },            
        labelTransform(d) {
            const x = (d.x0 + d.x1) / 2 * 180 / Math.PI;
            const y = (d.y0 + d.y1) / 2 * this.radius;
            return `rotate(${x - 90}) translate(${y},0) rotate(${x < 180 ? 0 : 180})`;
        },
        format(d) {
            return d3.format(d)
        },
        partition(data) {
            this.root = d3
                .hierarchy(data)
                .sum(() => 1)
                .sort((a, b) => b.value - a.value)
            return d3.partition().size([2 * Math.PI, this.root.height + 1])(this.root)
        },        
    },
}
</script>