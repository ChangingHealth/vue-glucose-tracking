<script>
import { defineComponent } from 'vue';
import GlucoseTracking from '@/glucose-tracking.vue';

export default defineComponent({
  name: 'ServeDev',
  components: {
    GlucoseTracking
  }
});
</script>

<template>
  <div id="app">
    <glucose-tracking />
  </div>
</template>
