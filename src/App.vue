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
import { circular, random } from 'graphology-layout';

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
      graph: Graph,
      hegemonyValues: {},
      bgpstateValues: [],
    }
  },
  methods: {
    async handleSubmit() {
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
        this.hegemonyValues[result.asn] = result.hege;
      })
      console.log(this.hegemonyValues);
    },
    async loadBgpData() {
      let timestamp = new Date(this.date).toISOString();
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

      let graphValues = {};

      this.bgpstateValues.forEach(route => {
        let prev_asn = "Internet";
        route.path.forEach(asn => {
          if (asn in this.hegemonyValues) {
            if (prev_asn != asn) {
              if (!(prev_asn in graphValues)) {
                graphValues[prev_asn] = {};
              }
              graphValues[prev_asn][asn] += 1;
              // Keep track of unmeasured hegemony
              // consumedHege[asn][prev_asn] = this.hegemonyValues[prev_asn];
            }
            prev_asn = asn;
          }
          else {
            prev_asn = 'Internet';
          }
        })
      })
      console.log(graphValues);
      

      this.graph = new Graph({
        multi: true,
        allowSelfLoops: true,
        type: "directed"
      });
      const rendrer = new Sigma(this.graph, container, {
        settings: {
          minArrowSize: 10,
        }
      });
      // adding the main node Internet.
      // this.graph.addNode('Internet', {
      //   size: 15, label: 'Internet',color:'green',
      // })
      random.assign(this.graph);
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
