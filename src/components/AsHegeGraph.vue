
<template>
    <div class="graph-container">        
        <p>plotting the graph for asn: 
            <span>{{ asNumber }}</span>
            and for date: 
            <span>{{ timestamp }}</span>
        </p>
        <div id="graph-holder">
            actual graph is here.
        </div>
    </div>
</template>

<script>
// import Sigma from "sigma";
// import Graph from "graphology";
export default {
    name: 'AsHegeGraph',
    props: {
        asNumber: { required: true, type: String },
        timestamp: {required: true, type: String },
    },
    data() {
        return {
            asn:this.asNumber,
            date : this.timestamp,         
        }
    },
    methods: {},
}

</script>

<style scoped>
.graph-container {
    background-color: beige;    
    color: black;
    margin-top: .5rem;
    padding: .25rem;
}
span{
    font-style: italic;
    font-weight: bold;
}
</style>
