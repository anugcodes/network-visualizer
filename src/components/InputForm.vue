<template>
    <form @submit.prevent="handleSubmit">        
        <label for="as-number">{{ asn_label }}</label>
        <input type="text" id="as-number" v-model="asn" />
        <label for="date-picker">{{ date_label }}</label>
        <input type="date" id="date-picker" v-model="date"/>
        <button type="submit">plot graph</button>
    </form>
</template>
  
<script>
export default {
    name: 'InputForm',
    data(){        
        return {
            date:new Date().toISOString().split('T')[0],
            asn:'',
            asn_label: "enter asn or ip",
            date_label: "enter timestamp"
        }
    },
    methods: {
        handleSubmit() {
            // e.preventDefault();
            console.log(this.date, this.asn);
        }
    }
};
</script>

<style>
form{
    width: 20rem;   
    padding:.5rem; 
    /* border : 1px solid white; */
    border-right: 2px dotted white;
}
input{
    width: 100%;
    padding: .5rem;
    margin-top: .5rem;    
    margin-bottom: .5rem;
}
button{
    padding: .5rem 1rem;
}
</style>
  