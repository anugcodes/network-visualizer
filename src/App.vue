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
        <button type="submit">plot</button>
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
import { circular } from 'graphology-layout';

const HEGE_BASE_URL = "https://ihr.iijlab.net/ihr/api/hegemony";
const BGPLAY_BASE_URL = "https://stat.ripe.net/data/bgplay/data.json";
const BGPSTATE_BASE_URL = "https://stat.ripe.net/data/bgp-state/data.json";


export default {
  name: "App",
  components: {
  },
  data() {
    return {
      date: new Date().toISOString().split('T')[0],
      asNumber: "",
      hegemonyValues: [],
      bgpstateValues: [],
    }
  },
  methods: {
    async handleSubmit() {
      console.log("values from form: ", this.asNumber, " --- ", this.date);
      console.log("calling api using axios...");
      await this.loadData();
      this.plotGraph();
    },

    // call hegemony api to load hegemony data.
    async loadData() {
      if (this.asNumber === '0' || this.asNumber === '') return;
      let timestamp = new Date(this.date).toISOString();
      console.log(timestamp);

      // for hegemony values.
      const hege_res = await axios.get(HEGE_BASE_URL, {
        params: {
          originasn: this.asNumber,
          timebin: timestamp,
        }
      })
      this.hegemonyValues = hege_res.data.results;

      // for bgp values using bgp state api
      const bgp_res = await axios.get(BGPSTATE_BASE_URL, {
        params: {
          resource: this.asNumber,
          timestamp: timestamp,
        }
      })
      this.bgpstateValues = bgp_res.data.data.bgp_state;
    },
    scaleNodeSize(hege, nodeminSize, nodemaxSize) {
      const hegeValue = Math.floor(hege * 100);
      if (hegeValue < nodeminSize) return nodeminSize;
      else if (hegeValue > nodemaxSize) return nodemaxSize;
      else return Math.floor(hege * 100);
    },

    plotGraph() {
      const container = this.$refs.graphHolder;
      const graph = new Graph({
        multi: true,
        allowSelfLoops: true,
        type: "directed"
      });
      const rendrer = new Sigma(graph, container, {
        settings: {
          minArrowSize: 10,
        }
      });      

      this.hegemonyValues.forEach(ele => {
        if (ele.originasn !== ele.asn) {
          graph.addNode(ele.asn, {
            size: this.scaleNodeSize(ele.hege, 10, 25), label: ele.asn, color: 'red',
          })
        }
        else {
          graph.addNode(ele.asn, {
            size: this.scaleNodeSize(ele.hege, 10, 25), label: ele.asn, color: 'green',
          })
        }
      }),
        this.hegemonyValues.forEach(ele => {
          graph.addEdge(ele.originasn, ele.asn, {
            type: 'arrow',
            size: 5,
          });
        })
      circular.assign(graph);
      rendrer.refresh();
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
