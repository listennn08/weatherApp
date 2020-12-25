<template>
  <div>
    {{ count }}
  </div>
</template>

<script lang="ts">
import { defineComponent } from 'vue';

export default defineComponent({
  name: 'InnerCount',
  props: {
    count: {
      type: Number,
    },
  },
  setup(props, context) {
    return {
      props,
      context,
    };
  },
});
</script>

<style scoped>

</style>
