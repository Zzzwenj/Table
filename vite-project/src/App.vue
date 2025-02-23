<template>
  <div id="app">
    <my-table />
  </div>
</template>

<script lang="ts">
import { defineComponent } from 'vue';
import MyTable from './components/MyTable.vue';

export default defineComponent({
  name: 'App',
  components: {
    MyTable,
  }
});
</script>

<style>
/* Global styles */
</style>
生成excel