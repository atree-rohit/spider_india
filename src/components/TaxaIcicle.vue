<style scoped>
</style>
<template>
    <div class=''>
        <v-breadcrumbs :items="breadcrumbs" divider=">" class="px-0 py-1">
            <template v-slot:item="{ item }">
                <v-breadcrumbs-item
                    class="breadcrumb-item"
                    @click.native="crumbClick(item.text)"
                    v-text="item.text"
                />
            </template>
        </v-breadcrumbs>
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
            focus: null,
            rect: null,
            cell: null,
            text: null,
            tspan: null,
            selected_taxa: ['Araneae'],



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
            watch_click: false,
            taxa_levels: ['Class', 'Order', 'Sub-Order', 'Super-Family', 'Family', 'Sub-Family', 'Tribe', 'Sub-Tribe', 'Genus', 'Species'],
            // taxa_levels: ["Super-family", "Family", "Sub-family", "Tribe", "Genus", "Species"],
        }
    },
    computed: {
        color() {
            return d3.scaleOrdinal(d3.quantize(d3.interpolateRainbow, this.speciesData.children.length +1))
        },
        breadcrumbs() {
            let op = []
            this.selected_taxa.map((t) => {
                op.push({
                    text: t,
                    disabled: false,
                })
            })
            return op
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
                // (d) => d.taxon_class_name,
                (d) => d.taxon_order_name,
                // (d) => d.taxon_suborder_name,
                (d) => d.taxon_superfamily_name,
                (d) => d.taxon_family_name,
                (d) => d.taxon_subfamily_name,
                (d) => d.taxon_tribe_name,
                // (d) => d.taxon_subtribe_name,
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
                op["value"] = 1
            }
            return op
        },
        partition(data) {
            const root = d3.hierarchy(data)
                .sum(d => d.value)
                .sort((a, b) => b.height - a.height || b.value - a.value);  
            return d3.partition()
                .size([this.height, (root.height + 1) * this.width / 3])(root);
        },
        chart() {
            this.focus = this.root;
            let format = d3.format(",d")
            let color = d3.scaleOrdinal(d3.quantize(d3.interpolateRainbow, this.speciesData.children.length + 1))

            if (!d3.select("#icicleChart svg").empty()) {
                d3.selectAll("#icicleChart svg").remove()
            }

            const svg = d3.select("#icicleChart")
                .append("svg")
                // .classed("bg-light text-center border boorder-primary rounded", true)
                .attr("viewBox", [0,0, this.width, this.height])

            this.cell = svg
                .selectAll("g")
                .data(this.root.descendants())
                .join("g")
                .attr("transform", d => `translate(${d.y0},${d.x0})`);

            this.rect = this.cell.append("rect")
                .attr("width", d => d.y1 - d.y0 - 1)
                .attr("height", d => this.rectHeight(d))
                .attr("fill-opacity", 0.6)
                .attr("fill", d => {
                    if (!d.depth) return "#ccc";
                    while (d.depth > 1) d = d.parent;
                    return color(d.data.name);
                })
                .style("cursor", "pointer")
                .on("click", this.clicked);

            this.text = this.cell.append("text")
                .style("user-select", "none")
                .attr("pointer-events", "none")
                .attr("x", 4)
                .attr("y", 13)
                .attr("fill-opacity", d => +this.labelVisible(d));

            this.text.append("tspan")
                .text(d => d.data.name);

            this.tspan = this.text.append("tspan")
                .attr("fill-opacity", d => this.labelVisible(d) * 0.7)
                .text(d => ` ${format(d.value)}`);

            this.cell.append("title")
                .text(d => `${d.ancestors().map(d => d.data.name).reverse().join("/")}\n${format(d.value)}`);
        },
        clicked(event, p) {
            
            this.focus = (this.focus === p) ? p = p.parent : p;
            
            
            if(p !== null ){
                this.selected_taxa = this.populate_breadcrumbs(p,[]);
                this.root.each(d => d.target = {
                    x0: (d.x0 - p.x0) / (p.x1 - p.x0) * this.height,
                    x1: (d.x1 - p.x0) / (p.x1 - p.x0) * this.height,
                    y0: d.y0 - p.y0,
                    y1: d.y1 - p.y0
                });
    
                const t = this.cell.transition().duration(750)
                    .attr("transform", d => `translate(${d.target.y0},${d.target.x0})`);
    
                this.rect.transition(t).attr("height", d => this.rectHeight(d.target));
                this.text.transition(t).attr("fill-opacity", d => +this.labelVisible(d.target));
                this.tspan.transition(t).attr("fill-opacity", d => this.labelVisible(d.target) * 0.7);
            }
        },
        rectHeight(d) {
            return d.x1 - d.x0 - Math.min(1, (d.x1 - d.x0) / 2);
        },
        labelVisible(d) {
            return d.y1 <= this.width && d.y0 >= 0 && d.x1 - d.x0 > 16;
        },
        populate_breadcrumbs(p,result){
            if(p.data != null && p.data.name != "Reset"){
                if(p.parent != null){
                    result = this.populate_breadcrumbs(p.parent, result);
                }
                result.push(p.data.name)
            }
            return result
        },
        crumbClick(text){
            this.root.descendants().forEach(a => {
                if(a.data.name == text){
                    this.clicked([], a)
                }
            })
        },
    },
}
</script>