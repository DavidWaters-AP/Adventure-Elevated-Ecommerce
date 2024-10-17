<template>
  <section>
    <div class="container px-4 py-32">
      <div class="flex flex-row gap-8">
        <NuxtLink
          class="p-6 border border-gray-900 rounded-md bg-white bg-opacity-55"
          exactActiveClass="button-active"
          to="/pro-configurator"
          >Pro Board</NuxtLink
        >
        <NuxtLink
          class="p-6 border border-gray-900 rounded-md bg-white bg-opacity-55"
          exactActiveClass="button-active"
          to="/max-configurator"
          >Max Board</NuxtLink
        >
        <NuxtLink
          class="p-6 border border-gray-900 rounded-md bg-white bg-opacity-55"
          exactActiveClass="button-active"
          to="/master-configurator"
          >Master Board</NuxtLink
        >
      </div>
    </div>
  </section>
</template>
<script setup lang="ts">
  import { watch, ref, onMounted } from "vue";
  import { useRoute } from "vue-router";

  const route = useRoute();
  const data = ref(null);

  watch(
    () => route.path,
    async () => {
      data.value = await useAsyncGql("getCompositeProduct", {
        slug: "212",
      });
    }
  );

  onMounted(async () => {
    data.value = await useAsyncGql("getCompositeProduct", {
      slug: "212",
    });
  });
</script>

<style scoped></style>
