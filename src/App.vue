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

export default {
  name: "App",
  components: {
  },
  data() {
    return {
      date: new Date().toISOString().split('T')[0],
      asNumber: "",
      hegemony_data: [],
    }
  },
  methods: {
    async handleSubmit() {
      console.log("values from form: ", this.asNumber, " --- ", this.date);
      console.log("calling api using axios...");
      await this.loadDate();
      this.plotGraph();
    },

    // call hegemony api to load hegemony data.
    async loadDate() {
      if (this.asNumber === '0') return;
      let timestamp = new Date(this.date).toISOString();
      console.log(timestamp);
      const res = await axios.get(HEGE_BASE_URL, {
        params: {
          originasn: this.asNumber,
          timebin: timestamp,
        }
      })
      this.hegemony_data = res.data.results;
    },

    plotGraph() {
      const graph = new Graph({
        multi: true,
        allowSelfLoops: true,
        type: "directed"
      });
      const container = this.$refs.graphHolder;
      const rendrer = new Sigma(graph, container, {
        settings: {
          minArrowSize: 10,
        }
      });

      this.hegemony_data.forEach(ele => {
        if (ele.originasn !== ele.asn) {
          graph.addNode(ele.asn, {
            size: Math.floor(ele.hege * 100), label: ele.asn, color: 'red',
          })
        }
        else {
          graph.addNode(ele.asn, {
            size: Math.floor(ele.hege * 50), label: ele.asn, color: 'green',
          })
        }
      }),
        this.hegemony_data.forEach(ele => {
          graph.addEdge(ele.originasn, ele.asn, {
            type: 'arrow',
            size: 5,
          });
        })
      circular.assign(graph);
      rendrer.refresh();
      console.log(rendrer);
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
