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
        <div class="info">
          <p>hover over the node/edges to display node/edge name or </p>
          <p>select the edge to get more info about the source and target nodes</p>
          <p>select/click a node to get more information about that node.</p>
        </div>
      </form>

      <!-- graph component -->
      <div class="graph-container">
        <p>plotting the graph for asn:
          <span>{{ asNumber }}</span>
          and for date:
          <span>{{ date }}</span>
        </p>
        <div class="node-edge-info" ref="nodeEdgeInfoContainer"></div>
        <div id="graph-holder" ref="graphHolder"></div>
      </div>
      <div class="note">
        <h4>Note: </h4>This is simple VueJs component demonstrating how to fetch data from the IHR's Hegemony and bgplay
        api and plotting the graph after generating all the nodes(AS) and edges/relationships. In this example the
        hegemony data is fetched only for a timestamp value. and using the bgplay's initial state values relationships are
        generated.
      </div>
    </div>
  </div>
</template>

<script>
import axios from 'axios';
import Sigma from "sigma";
import Graph from "graphology";


const HEGE_BASE_URL = "https://ihr.iijlab.net/ihr/api/hegemony";
const BGPLAY_BASE_URL = "https://stat.ripe.net/data/bgplay/data.json";

// network request timeout
const timeout = 10000;

export default {
  name: "App",
  components: {
  },
  data() {
    return {
      date: new Date().toISOString().split('T')[0],
      asNumber: "",
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
      try {
        const hege_res = await axios.get(HEGE_BASE_URL, {
          params: {
            originasn: this.asNumber,
            timebin: timestamp,
          },
          timeout: timeout,
        })
        hege_res.data.results.forEach(result => {
          this.hegemonyValues[result.asn] = { asn: result.asn, name: result.asn_name, hege: result.hege };
        })
        console.log(this.hegemonyValues);
      } catch (err) {
        console.log(err.message, err.code);
      }
    },
    async loadBgpData() {
      let timestamp = new Date(this.date).toISOString();
      // for bgp values using bgp state api
      try {
        const bgp_res = await axios.get(BGPLAY_BASE_URL, {
          params: {
            resource: this.asNumber,
            starttime: timestamp,
          },
          timeout: timeout,
        })
        this.bgpstateValues = bgp_res.data.data.initial_state;
      } catch (err) {
        console.log(err.message, err.code);
      }
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

    nodeEdgeInfoTemplate(resource) {
      return `        
        <p>As Number: <span>${this.hegemonyValues[resource]['asn']}</span></p>
        <p>As Name: <span> ${this.hegemonyValues[resource]['name']} </span></p>
        <p>Hegemony Value: <span> ${this.hegemonyValues[resource]['hege']} </span></p>
      `
    },

    displayInfo(resourcekey) {
      this.$refs.nodeEdgeInfoContainer.innerHTML = "";
      if (resourcekey.search('n') === 0) {
        let node = resourcekey.replace('n-', '');
        this.$refs.nodeEdgeInfoContainer.innerHTML = `<h5>Node Information:</h5>` + this.nodeEdgeInfoTemplate(node);
      }
      if (resourcekey.search('e') === 0) {
        let source = resourcekey.replace('e', '').split('-')[0];
        let target = resourcekey.replace('e', '').split('-')[1];
        console.log(source, " ", target);
        this.$refs.nodeEdgeInfoContainer.innerHTML = `
        <h4>Edge Information:</h4>
        <h5>Source As</h5>
        ${this.nodeEdgeInfoTemplate(source)}
        <br>
        <h5>Target As</h5>
        ${this.nodeEdgeInfoTemplate(target)}
        <h5>Netwok Delay Information </h5>
        `
      }
    },


    plotGraph() {
      let graphValues = {};
      let consumedHege = {};
      let graph = new Graph({
        multi: true,
        allowSelfLoops: true,
        type: "directed"
      })

      // adding Internet node to hegemonyValues
      this.hegemonyValues['Internet'] = { asn: "Internet", name: "Internet", hege: 1.0 };
      Object.keys(this.hegemonyValues).forEach(asn => consumedHege[asn] = {});

      this.bgpstateValues.forEach(route => {
        let prev_asn = "Internet";
        route.path.forEach(asn => {
          if (asn in this.hegemonyValues) {
            if (prev_asn != asn) {
              if (!(prev_asn in graphValues)) {
                graphValues[prev_asn] = this.newCounter();
              }
              graphValues[prev_asn][asn] += 1;
              consumedHege[asn][prev_asn] = this.hegemonyValues[prev_asn]['hege']
            }
            prev_asn = asn;
          }
          else {
            prev_asn = 'Internet';
          }
        })
      })
      console.log(graphValues);


      

      Object.keys(this.hegemonyValues).forEach(asn => {
        if (asn === this.asNumber) {
          graph.addNode('n-' + asn, {
            x: 0.0,
            y: 0.5,
            label: asn,
            size: 12,
            color: 'blue',
          })
        }
        else if (asn === "Internet") {
          graph.addNode('n-' + asn, {
            x: 1,
            y: 0.5,
            label: asn,
            size: 12,
            color: 'green',
          })
        }
        else {
          let u = 0, v = 0;
          while (u < 0.1 || u > 0.9) u = Math.random();
          while (v === 0) v = Math.random();
          graph.addNode('n-' + asn, {
            x: u,
            y: v,
            label: asn,
            size: 12,
            color: 'gray',
          })
        }
      })

      Object.keys(graphValues).forEach(target => {
        let total_count = Object.values(graphValues[target]).reduce((a, b) => a + b, 0);        
        Object.keys(graphValues[target]).forEach(source => {
          let weight = this.hegemonyValues[target]['hege'] * (graphValues[target][source] / total_count);
          // calculate weight if the target node is rest of Internet.
          if (target == 'Internet') {
            var sum = 0;
            for (const [asn, hege] of Object.entries(consumedHege[source])) if (asn != target) sum += hege;
            // console.log(sum)
            weight = Math.abs(this.hegemonyValues[source]['hege'] - sum);            
          }
          // constrain the thickness of edges.
          console.log(weight*10);
          if( weight*10 < 1) weight = 1;
          else weight = weight*10;     
          // finally adding the edges to the graph object. 
          graph.addEdgeWithKey('e' + source + '-' + target, 'n-' + source, 'n-' + target, {           
            type: 'arrow',
            size: weight,
          })
        })
      })

      const container = this.$refs.graphHolder;
      let hoveredEdge = null;

      // getting the Sigma js Rendrer
      var renderer = new Sigma(graph, container, {
        // sigmajs settings 
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
        graph.setEdgeAttribute(edge, 'label', 'network delay for ' + graph.getEdgeAttribute(edge, 'size'));
        hoveredEdge = edge;
        renderer.refresh();
      });
      renderer.on("leaveEdge", ({ edge }) => {
        graph.setEdgeAttribute(edge, 'label', '');
        hoveredEdge = edge;
        renderer.refresh();
      });
      renderer.on("downEdge", ({ edge }) => {
        console.log(edge, "downEdge event");
        this.displayInfo(edge);
        hoveredEdge = edge;
        renderer.refresh();
      });

      // node events and functions
      renderer.on("enterNode", ({ node }) => {
        let asn = node.replace('n-', '');
        graph.updateNodeAttribute(node, 'label', n => n + " " + this.hegemonyValues[asn]['name']);
        renderer.refresh();
      });
      renderer.on("leaveNode", ({ node }) => {
        graph.setNodeAttribute(node, 'label', node.replace('n-', ''));
        renderer.refresh();
      });
      renderer.on("downNode", ({ node }) => {
        this.displayInfo(node);
        renderer.refresh();
      });



      // All the events from Sigma js
      // const nodeEvents = [
      //   "enterNode",
      //   "leaveNode",
      //   "downNode",
      //   "clickNode",
      //   "rightClickNode",
      //   "doubleClickNode",
      //   "wheelNode",
      // ];
      // const edgeEvents = ["downEdge", "clickEdge", "rightClickEdge", "doubleClickEdge", "wheelEdge"];
      // const stageEvents = ["downStage", "clickStage", "doubleClickStage", "wheelStage"];

      this.loading = false;
    },

    calculateAvgHegemony(data) {
      // data is the Array containing objects for each asn for different timestamps
      let hegeValues = {};
      let AvgHegeValues = {};
      data.forEach(node => {
        if (!(node.asn in hegeValues)) {
          hegeValues[node.asn] = {
            sum: node.hege,
            count: 1,
            name: node.asn_name,
          }
        } else {
          hegeValues[node.asn][sum] += node.hege;
          hegeValues[node.asn][count] += 1;
        }
      })
      Object.keys(hegeValues).forEach(asn => {
        AvgHegeValues[asn] = {
          avg_hege: hegeValues[asn]["sum"] / hegeValues[asn]["count"],
          name: hegeValues[asn]["name"],
        }
      })
      return AvgHegeValues;
    }


  },
};
</script>


<style scoped>
.container {
  padding: .5rem;
}

@media screen and (max-width: 600px) {
  .container {
    display: block;
  }

  .info {
    display: none;
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
  position: relative;
}

.form-control {
  margin: 0 .5rem;
  display: flex;
  flex-direction: column;
}

.form-control>input {
  width: 8rem;
}

form button {
  padding: 0 .5rem;
}

.graph-container {
  position: relative;
  background-color: beige;
  color: black;
  margin-top: .5rem;
  padding: .25rem;
}

.graph-container p span {
  font-style: italic;
  font-weight: bold;
}

#graph-holder {
  height: 32rem;
}

.node-edge-info {
  position: absolute;
  top: 0;
  right: 0;
  background-color: bisque;
  opacity: .5;
  height: 100%;
  width: 15rem;
}

.note {
  font-size: small;
}

.info {
  align-self: flex-end;
  position: absolute;
  right: 0;
  top: 0;
  margin: .5rem;
}

.info p {
  font-size: x-small;
  font-weight: 500;
  color: darkslateblue;
}
</style>



