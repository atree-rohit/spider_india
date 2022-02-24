<style scoped>
    #radialDendrogram{
        border: 1px dashed red;
        border-radius:100%
    }
</style>
<template>
    <div>
        <div id="radialDendrogram" />
    </div>
</template>

<script>
import * as d3 from "d3";
export default {
    name: "TaxaRadialDendrogram",
    props: ["data", "selected", "popup"],
    emits: ["taxaSelected"],
    data() {
        return {
            width: 600,
            height: 600,
            radius: 100,
            tree_data: null,
            taxa_levels: ['Class', 'Order', 'Sub-Order', 'Super-Family', 'Family', 'Sub-Family', 'Tribe', 'Sub-Tribe', 'Genus', 'Species'],
        }
    },
    computed: {
        tree() {
            return d3.tree()
                .size([2 * Math.PI, this.radius])
                .separation((a, b) => (a.parent == b.parent ? 1 : 2) / a.depth)
        }
    },
    watch: {
    },
    mounted() {
        this.init()
    },
    methods: {
        init() {
            let groups = d3.groups(
                this.data.slice(0,100),
                // d => d.taxon_class_name,
                // d => d.taxon_order_name,
                // d => d.taxon_suborder_name,
                // d => d.taxon_superfamily_name,
                d => d.taxon_family_name,
                d => d.taxon_subfamily_name,
                d => d.taxon_tribe_name,
                d => d.taxon_subtribe_name,
                d => d.taxon_genus_name,
                d => d.taxon_species_name,
            )
            this.tree_data = d3.hierarchy(this.hierarchy(groups))
            
            
            console.clear()
            
            console.log("Final", this.tree_data)
            this.chart()
            
        },
        hierarchy(array){
            let a = array[0]
            let op = { name: a[0] }            
            if(a[1][0].id === undefined){
                op["children"] = a[1].map((c) => this.hierarchy([c]))
            } else {
                // op["size"] = a[1].length
                // op["size"] = 1
            }
            return op
        },
        chart() {
            const root = this.tree(this.tree_data);
            // const svg = d3.create("svg");
            if (!d3.select("#radialDendrogram svg").empty()) {
                d3.selectAll("#radialDendrogram svg").remove()
            }

            
            const svg = d3.select("#radialDendrogram")
                .append("svg")
                .attr("preserveAspectRatio", "xMinYMin meet")
                // .attr("width", this.width)
                // .attr("height", this.height)
                .attr("viewBox", [-this.width / 2,-this.height / 2,this.width,this.height])
            
            svg.append("g")
                .attr("fill","none")
                .attr("stroke", "#555")
                .attr("stroke-opacity", 0.4)
                .attr("stroke-width", 1.5)
                .selectAll("path")
                .data(root.links())
                    .join("path")
                    .attr("d", d3.linkRadial()
                    .angle(d => d.x)
                    .radius(d => d.y));
  
            svg.append("g")
                .selectAll("circle")
                .data(root.descendants())
                .join("circle")
                    .attr("transform", d => `
                        rotate(${d.x * 180 / Math.PI - 90})
                        translate(${d.y},0)
                    `)
                    .attr("fill", d => d.children ? "#555" : "#999")
                    .attr("r", 2.5);

            svg.append("g")
                .attr("font-family", "sans-serif")
                .attr("font-size", 10)
                .attr("stroke-linejoin", "round")
                .attr("stroke-width", 3)
                .selectAll("text")
                .data(root.descendants())
                .join("text")
                    .attr("transform", d => `
                        rotate(${d.x * 180 / Math.PI - 90}) 
                        translate(${d.y},0) 
                        rotate(${d.x >= Math.PI ? 180 : 0})
                    `)
                    .attr("dy", "0.31em")
                    .attr("x", d => d.x < Math.PI === !d.children ? 6 : -6)
                    .attr("text-anchor", d => d.x < Math.PI === !d.children ? "start" : "end")
                    .text(d => d.data.name)
                    .clone(true).lower()
                    .attr("stroke", "white");

            // return svg.attr("viewBox", this.autoBox).node();
        },
        autoBox() {
            document.body.appendChild(this);
            const {x, y, width, height} = this.getBBox();
            document.body.removeChild(this);
            return [x, y, width, height];
        },
    },
}
</script>