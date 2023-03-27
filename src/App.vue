<template>
  <div>
    <h2>Network Dependency visualizer using Sigma js</h2>
    <div class="container">
      <InputForm />
      <AsHegeGraph as-number="string" timestamp="timestamp" />
    </div>
  </div>
</template>

<script>
import AsHegeGraph from './components/AsHegeGraph.vue';
import InputForm from './components/InputForm.vue';
export default {
  name: "App",
  data() {
    return {}
  },
  methods: {},
  components: {
    AsHegeGraph,
    InputForm,
  }
};
</script>


<style scoped>
.container {
  display: flex;
  width: 100%;
  justify-content: space-between;
  padding: .5rem;
}
@media screen and (max-width: 1000px) {
  .container{
    display: block;
  }
}

h2 {
  text-align: center;
  color: whitesmoke;
  padding: 1rem;
}
</style>
