<template>
  <div>
    <h2>Network Dependency visualizer using Sigma js</h2>
    <div class="container">
      <!-- form for values to plot in graph: -->
      <form @submit.prevent="handleSubmit">
        <div class="form-control">
          <label for="asn">enter asn: </label>
          <input type="text" required v-model="asNumber" />
        </div>
        <div class="form-control">
          <label for="asn">select date: </label>
          <input type="date" v-model="date"/>
        </div>
        <button type="submit" >plot</button>
      </form>
      <!-- graph component -->
      <AsHegeGraph :as-number="asNumber" :timestamp="date" />
    </div>
  </div>
</template>

<script>
import AsHegeGraph from './components/AsHegeGraph.vue';


export default {
  name: "App",
  components: {
    AsHegeGraph,
  },
  data() {
    return {
      date: new Date().toISOString().split('T')[0],
      asNumber: "",
    }
  },
  methods: {
    handleSubmit(){
      console.log("values from form: ",this.asNumber," --- ",this.date)
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
  color: whitesmoke;
  padding: .5rem;
}
form{
  display: flex;
  align-items: flex-end;
  border-bottom: 2px dotted white;
  padding: .5rem 0;
}
.form-control{
  margin: 0 .5rem;
  display: flex;
  flex-direction: column;
}
input{
  width: 8rem;
}
button{
  padding: 0 .5rem;
}
</style>
