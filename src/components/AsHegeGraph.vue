
<template>
    <div class="graph-container">
        graph goes here.
        <div id="graph-holder">
            actual graph is here.
        </div>
    </div>
</template>

<script>
import Sigma from "sigma";
import Graph from "graphology";
export default {
    name: 'AsHegeGraph',
    props: {
        asNumber: { required: true, type: String },
        timestamp: {required: true, type: String },
        timestamp__lte: {required: false, type: String },
        timestamp__gte: {required: false, type: String },
    },
    data() {
        return {

        }
    },
    methods: {},
}

</script>

<style scoped>
.graph-container {
    width: 100%;
    margin: .5rem;
    padding: .5rem;
    background-color: honeydew;
    opacity: .8;
    color: black;
}
</style>
