<template>
  <div>
    <div class="container">
      <h2>Network Dependency visualizer using Sigma js</h2>
      <!-- form for values to plot in graph: -->
      <form @submit.prevent="handleSubmit">
        <div class="form-control">
          <label for="asn">enter asn: </label>
          <input type="text" required v-model="asNumber" />
        </div>
        <div class="form-control">
          <label for="asn">select date: </label>
          <input type="date" v-model="date" />
        </div>
        <button v-if="!loading" type="submit">plot</button>
        <p v-if="loading">loading...</p>
      </form>

      <!-- graph component -->
      <div class="graph-container">
        <p>plotting the graph for asn:
          <span>{{ asNumber }}</span>
          and for date:
          <span>{{ date }}</span>
        </p>
        <!-- this div will be used to plot graph -->
        <div id="graph-holder" ref="graphHolder"></div>
      </div>

    </div>
  </div>
</template>

<script>
import axios from 'axios';
import Sigma from "sigma";
import Graph from "graphology";
import { circular, random } from 'graphology-layout';


const HEGE_BASE_URL = "https://ihr.iijlab.net/ihr/api/hegemony";
const BGPLAY_BASE_URL = "https://stat.ripe.net/data/bgplay/data.json";


export default {
  name: "App",
  components: {
  },
  data() {
    return {
      date: new Date().toISOString().split('T')[0],
      asNumber: "",
      graph: Graph,
      hegemonyValues: {},
      bgpstateValues: [],
      loading: false,
    }
  },
  methods: {
    async handleSubmit() {
      this.loading = true;
      console.log("values from form: ", this.asNumber, " --- ", this.date);
      if (this.asNumber === '0' || this.asNumber === '') return;
      console.log("calling api using axios...");
      await this.loadHegemonyData();
      await this.loadBgpData();
      this.plotGraph();
    },
    async loadHegemonyData() {
      let timestamp = new Date(this.date).toISOString();
      const hege_res = await axios.get(HEGE_BASE_URL, {
        params: {
          originasn: this.asNumber,
          timebin: timestamp,
        }
      })
      hege_res.data.results.forEach(result => {
        this.hegemonyValues[result.asn] = { asn: result.asn, name: result.asn_name, hege: result.hege };
      })
      console.log(this.hegemonyValues);
    },
    async loadBgpData() {
      let timestamp = new Date(this.date).toISOString();
      // for bgp values using bgp state api
      const bgp_res = await axios.get(BGPLAY_BASE_URL, {
        params: {
          resource: this.asNumber,
          starttime: timestamp,
        }
      })
      this.bgpstateValues = bgp_res.data.data.initial_state;
    },


    newCounter() {
      let handler = {
        get: function (target, name) {
          return target.hasOwnProperty(name) ? target[name] : 0;
        }
      };
      let emptyObj = {};
      return new Proxy(emptyObj, handler);
    },

    plotGraph() {
      let graphValues = {};
      let trace = {
        nodes: [],
        edges: [],
      }
      this.bgpstateValues.forEach(route => {
        let prev_asn = "Internet";
        route.path.forEach(asn => {
          if (asn in this.hegemonyValues) {
            if (prev_asn != asn) {
              if (!(prev_asn in graphValues)) {
                graphValues[prev_asn] = this.newCounter();
              }
              graphValues[prev_asn][asn] += 1;
            }
            prev_asn = asn;
          }
          else {
            prev_asn = 'Internet';
          }
        })
      })
      console.log(graphValues);


      // adding Internet node to hegemonyValues
      this.hegemonyValues['Internet'] = { asn: "Internet", name: "Internet", hege: 1.0 };

      Object.keys(this.hegemonyValues).forEach(asn => {
        if (asn === this.asNumber) {
          trace.nodes.push({
            key: 'n-' + asn,
            attributes: {
              x: Math.random(),
              y: Math.random(),
              label: asn,
              size: 12,
              color: 'blue',
            },
          })
        }
        else if (asn === "Internet") {
          trace.nodes.push({
            key: 'n-' + asn,
            attributes: {
              x: Math.random(),
              y: Math.random(),
              label: asn,
              size: 12,
              color: 'green',
            },
          })
        }
        else {
          trace.nodes.push({
            key: 'n-' + asn,
            attributes: {
              x: Math.random(),
              y: Math.random(),
              label: asn,
              size: 12,
              color: 'gray',
            },
          })
        }
      })

      Object.keys(graphValues).forEach(asn => {
        Object.keys(graphValues[asn]).forEach(source => {
          trace.edges.push({
            key: 'e' + source + '-' + asn,
            source: 'n-' + source,
            target: 'n-' + asn,
            attributes: {
              color: '',
              type: 'arrow',
              size: 2,
            }
          })
        })
      })
      console.log(trace);

      // creating the graph object using Graph from graphology
      let graph = new Graph({
        multi: true,
        allowSelfLoops: true,
        type: "directed"
      })
      // setting the graph from the trace data
      graph.import(trace);
      random.assign(graph);
      const container = this.$refs.graphHolder;
      let hoveredEdge = null;

      // getting the Sigma js Rendrer
      var renderer = new Sigma(graph, container, {
        allowInvalidContainer: true,
        hideEdgesOnMove: true,
        edgeLabelSize: 12,
        minArrowSize: 10,
        renderEdgeLabels: true,
        enableEdgeClickEvents: true,
        enableEdgeWheelEvents: true,
        enableEdgeHoverEvents: "debounce",
        edgeReducer(edge, data) {
          const res = { ...data };
          if (edge === hoveredEdge) res.color = "#cc0000";
          return res;
        },
      });


      // edge events and functions
      renderer.on("enterEdge", ({ edge }) => {
        graph.setEdgeAttribute(edge, 'label', 'network delay for ' + edge);
        hoveredEdge = edge;
        renderer.refresh();
      });
      renderer.on("leaveEdge", ({ edge }) => {
        graph.setEdgeAttribute(edge, 'label', '');
        hoveredEdge = null;
        renderer.refresh();
      });
      renderer.on("downEdge", ({ edge }) => {
        console.log(edge, "downEdge event");        
        hoveredEdge = null;
        renderer.refresh();
      });

      // node events and functions
      renderer.on("enterNode", ({ node }) => {
        console.log(node, "enterNode event");  
        graph.updateNodeAttribute(node,'label',n => n+" AS name");         
        renderer.refresh();
      });
      renderer.on("leaveNode", ({ node }) => {
        console.log(node, "enterNode event");  
        graph.setNodeAttribute(node,'label',node.replace('n-',''));       
        renderer.refresh();
      });



      // All the events from Sigma js
      const nodeEvents = [
        "enterNode",
        "leaveNode",
        "downNode",
        "clickNode",
        "rightClickNode",
        "doubleClickNode",
        "wheelNode",
      ];
      const edgeEvents = ["downEdge", "clickEdge", "rightClickEdge", "doubleClickEdge", "wheelEdge"];
      const stageEvents = ["downStage", "clickStage", "doubleClickStage", "wheelStage"];



      this.loading = false;
    },
  },
};
</script>


<style scoped>
.container {
  padding: .5rem;
}

@media screen and (max-width: 800px) {
  .container {
    display: block;
  }
}

h2 {
  font-size: 1.25rem;
  padding: .5rem;
  border-bottom: 1px solid black;
}

form {
  display: flex;
  align-items: flex-end;
  padding: .5rem 0;
}

.form-control {
  margin: 0 .5rem;
  display: flex;
  flex-direction: column;
}

input {
  width: 8rem;
}

button {
  padding: 0 .5rem;
}

.graph-container {
  background-color: beige;
  color: black;
  margin-top: .5rem;
  padding: .25rem;
}

span {
  font-style: italic;
  font-weight: bold;
}

#graph-holder {
  height: 32rem;
}
</style>



