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

        <h1>{{ data?.value?.name }}</h1>
      </div>
    </div>
  </section>
</template>

<script setup lang="ts">
  const { data } = await useAsyncGql("getCompositeProduct", {
    slug: "209",
  });
</script>

<style scoped>
  .button-active {
    @apply bg-slate-500;
    @apply text-white;
  }
</style>
