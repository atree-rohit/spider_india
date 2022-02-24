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
        <div id="treeOfLifeChart">
        </div>        
    </div>
</template>

<script>
import * as d3 from "d3";
export default {
    name: "TreeOfLife",
    props: ["data", "selected", "popup"],
    emits: ["taxaSelected"],
    data() {
        return {
            width: 1200,
            height: 800,
            speciesData: null,
            selected_taxa: ['Araneae'],
            outerRadius: 477,
            innerRadius:307,
            newickData: null,
            chart_mode: "order",
            color_col: 8,
            taxon_cols: ["taxon_class_name","taxon_order_name","taxon_suborder_name","taxon_superfamily_name","taxon_family_name","taxon_subfamily_name","taxon_tribe_name","taxon_subtribe_name","taxon_genus_name","taxon_species_name"],
        }
    },
    computed: {
        color() {
            return d3.scaleOrdinal()
                // .domain(["Bacteria", "Eukaryota", "Archaea"])
                .domain([...new Map(this.speciesData.map(item =>
                                    [item[this.taxon_cols[4]], item]))].map((o) => o[0]).filter((n) => n.length > 0))
                .range(d3.schemeCategory10)
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
        cluster(){
            return d3.cluster()
                .size([360, this.innerRadius])
                .separation(() => 1)
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
            this.width = 954
            this.height = 1200
        },
        init_data() {
            console.clear()
            this.speciesData = [...new Map(this.data.map(item =>
                                    [item.taxon_species_name, item])).values()]            
            
            const rollups = (data, ...keys) => {
                return d3.rollups(
                    data,
                    () => "",
                    ...keys.map(k => d => d[k])
                )
            }
            const end = this.taxon_cols.indexOf(this.taxon_cols[this.color_col]) + 1
            const start = 0
            
            const array = rollups(this.speciesData, ...this.taxon_cols.slice(start,end))

            console.log(array)
            this.newickData = this.newickHierarchy(array[0]).replaceAll("()", "") + ";"
            
        },
        newickHierarchy(a){
            let op = ""
            let inner = ""
            if(!Array.isArray(a[0])){
                if(Array.isArray(a[1])){
                    inner = a[1].map((o) => this.newickHierarchy(o)).join(",")
                }
                op = `(${inner})${a[0].replaceAll(" ", "_")}`
            }
            return op
        },
        chart() {
            const root = d3.hierarchy(this.parseNewick(this.newickData), d => d.branchset)
              .sum(d => d.branchset ? 0 : 1)
              .sort((a, b) => (a.value - b.value) || d3.ascending(a.data.length, b.data.length));
        
            this.cluster(root);
            this.setRadius(root, root.data.length = 0, this.innerRadius / this.maxLength(root));
            this.setColor(root);
    

            if (!d3.select("#treeOfLifeChart svg").empty()) {
                d3.selectAll("#treeOfLifeChart svg").remove()
            }

            const svg = d3.select("#treeOfLifeChart")
                .append("svg")
                .attr("viewBox", [-this.outerRadius, -this.outerRadius, this.width, this.width])
                .attr("font-family", "sans-serif")
                .attr("font-size", 10);
    
            svg.append("g")
            .call(this.legend);
            
            svg.append("style").text(`
                .link--active {
                    stroke: #000 !important;
                    stroke-width: 1.5px;
                }
                .link-extension--active {
                    stroke-opacity: .6;
                }
                .label--active {
                    font-weight: bold;
                }
            `);
  
            const linkExtension = svg.append("g")
                .attr("fill", "none")
                .attr("stroke", "#000")
                .attr("stroke-opacity", 0.25)
                .selectAll("path")
                .data(root.links().filter(d => !d.target.children))
                .join("path")
                  .each(function(d) { d.target.linkExtensionNode = this; })
                  .attr("d", this.linkExtensionConstant);

            const link = svg.append("g")
                .attr("fill", "none")
                .attr("stroke", "#000")
                .selectAll("path")
                .data(root.links())
                .join("path")
                  .each(function(d) { d.target.linkNode = this; })
                  .attr("d", this.linkConstant)
                  .attr("stroke", d => d.target.color);

            svg.append("g")
                .selectAll("text")
                .data(root.leaves())
                .join("text")
                    .attr("dy", ".31em")
                    .attr("transform", d => `rotate(${d.x - 90}) translate(${this.innerRadius + 4},0)${d.x < 180 ? "" : " rotate(180)"}`)
                    .attr("text-anchor", d => d.x < 180 ? "start" : "end")
                    .text(d => d.data.name.replace(/_/g, " "))
                    .on("mouseover", mouseovered(true))
                    .on("mouseout", mouseovered(false))
                    .on("click", this.reCenterTree)
            
            function update(checked) {
                const t = d3.transition().duration(750);
                linkExtension.transition(t).attr("d", checked ? this.linkExtensionVariable : this.linkExtensionConstant);
                link.transition(t).attr("d", checked ? this.linkVariable : this.linkConstant);
            }
            
            function mouseovered(active) {
                return function(event, d) {
                    d3.select(this).classed("label--active", active);
                    d3.select(d.linkExtensionNode).classed("link-extension--active", active).raise();
                    do d3.select(d.linkNode).classed("link--active", active).raise();
                    while (d == d.parent);
                };
            }
            
            return Object.assign(svg.node(), {update});
        },
        reCenterTree(n, d){
            console.clear()
            
            this.color_col = (this.color_col < 4) ? this.color_col + d.depth : 8
            
            this.init_data()
            this.chart()
        },
        maxLength(d) {
            return d.data.length + (d.children ? d3.max(d.children, this.maxLength) : 0);
        },
        setRadius(d, y0, k) {
            d.radius = (y0 += d.data.length) * k;
            if (d.children) d.children.forEach(d => this.setRadius(d, y0, k));
        },
        setColor(d) {
            var name = d.data.name;
            d.color = this.color.domain().indexOf(name) >= 0 ? this.color(name) : d.parent ? d.parent.color : null;
            if (d.children) d.children.forEach(this.setColor);
        },
        linkVariable(d) {
            return this.linkStep(d.source.x, d.source.radius, d.target.x, d.target.radius);
        },
        linkConstant(d) {
            return this.linkStep(d.source.x, d.source.y, d.target.x, d.target.y);
        },
        linkExtensionVariable(d) {
            return this.linkStep(d.target.x, d.target.radius, d.target.x, this.innerRadius);
        },
        linkExtensionConstant(d) {
            return this.linkStep(d.target.x, d.target.y, d.target.x, this.innerRadius);
        },
        linkStep(startAngle, startRadius, endAngle, endRadius) {
            const c0 = Math.cos(startAngle = (startAngle - 90) / 180 * Math.PI);
            const s0 = Math.sin(startAngle);
            const c1 = Math.cos(endAngle = (endAngle - 90) / 180 * Math.PI);
            const s1 = Math.sin(endAngle);
            return "M" + startRadius * c0 + "," + startRadius * s0 + (endAngle === startAngle ? "" : "A" + startRadius + "," + startRadius + " 0 0 " + (endAngle > startAngle ? 1 : 0) + " " + startRadius * c1 + "," + startRadius * s1) + "L" + endRadius * c1 + "," + endRadius * s1;
        },
        legend(svg) {
            const g = svg
                .selectAll("g")
                .data(this.color.domain())
                .join("g")
                    .attr("transform", (d, i) => `translate(${-this.outerRadius},${-this.outerRadius + i * 20})`);
            
            g.append("rect")
                .attr("width", 18)
                .attr("height", 18)
                .attr("fill", this.color);
  
            g.append("text")
                .attr("x", 24)
                .attr("y", 9)
                .attr("dy", "0.35em")
                .text(d => d);    
        },
        parseNewick(a){
            for(var e=[],r={},s=a.split(/\s*(;|\(|\)|,|:)\s*/),t=0 ; t<s.length ; t++){
                var n=s[t];
                var c = null
                switch(n){
                    case"(": c={};
                        r.branchset=[c],e.push(r),r=c;
                        break;
                    case",": c={};
                        e[e.length-1].branchset.push(c),r=c;
                        break;
                    case")":r=e.pop();
                        break;
                    case":":break;
                    default:var h=s[t-1];
                        (")"==h||"("==h||","==h) ? r.name=n : ":"==h&&(r.length=parseFloat(n))
                }
            }
            return r
        },
    },
}
</script>