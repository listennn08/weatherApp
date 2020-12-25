<template>
  <div>
    {{ count }}
    <button @click="add">add</button>
    <Inner-Count v-model="count" />
  </div>
</template>

<script lang="ts">
import { ref, defineComponent } from 'vue';
import InnerCount from '@/components/VInnerCount.vue';

export default defineComponent({
  name: 'Count',
  components: {
    InnerCount,
  },
  setup() {
    const count = ref(0);

    const add = (): void => {
      count.value += 1;
    };

    return {
      count,
      add,
    };
  },
});
</script>

<style scoped>

</style>
