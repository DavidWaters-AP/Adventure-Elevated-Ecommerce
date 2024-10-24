<template>
  <div>
    <div class="mx-auto container px-4 py-32">
      <div class="flex flex-row gap-12">
        <div class="w-1/2">
          <img :src="product.product?.image.sourceUrl" alt="product image" />
        </div>
        <div class="w-1/2">
          <h1 class="text-3xl font-bold mb-4">{{ product.product?.name }}</h1>
          <div class="mb-12" v-html="product.product?.description"></div>
          <div>
            <div
              class="mb-16"
              v-for="component in product.product?.components"
              :key="component.id"
            >
              <h3 class="text-xl mb-4 font-semibold">
                {{ component.title }}
              </h3>
              <div class="flex flex-row gap-4">
                <div
                  v-for="v in component.queryOptions[0].variations.nodes"
                  :key="v.id"
                >
                  <label
                    :class="[
                      'flex rounded-md border border-solid p-2 h-full w-32 flex-col items-center cursor-pointer',
                      {
                        'border-blue-500 bg-blue-100':
                          selectedValues[component.id] ===
                          v.attributes.nodes[0].value,
                        'border-gray-200 bg-gray-100':
                          selectedValues[component.id] !==
                          v.attributes.nodes[0].value,
                      },
                    ]"
                  >
                    <input
                      class="hidden"
                      type="radio"
                      :name="`component-${component.id}`"
                      :value="v.attributes.nodes[0].value"
                      v-model="selectedValues[component.id]"
                    />
                    <NuxtImg
                      class="mb-4 w-full object-contain"
                      :src="v.featuredImage.node.sourceUrl"
                      alt="product image"
                      width="50"
                      height="50"
                    />
                    <span class="text-md font-semibold">
                      {{
                        v.attributes.nodes.map((attr) => attr.value).join(", ")
                      }}
                    </span>
                    <span class="text-sm">{{ v.price }}</span>
                  </label>
                </div>
              </div>
              <div class="mt-4 font-semibold">
                You've Selected:
                <span class="font-thin">{{
                  selectedValues[component.id]
                }}</span>
              </div>
            </div>
          </div>
          <div
            class="fixed bottom-0 bg-opacity-60 bg-slate-900 w-full left-0 backdrop-blur-lg text-white"
          >
            <div
              class="container mx-auto py-6 px-4 flex flex-row justify-between items-center"
            >
              <p class="text-3xl font-thin">
                <span class="font-semibold">Total Amount: </span
                >{{ priceFormatter.format(configAmount) }}
              </p>
              <button
                class="bg-red-400 px-12 py-4 rounded-full hover:bg-white transition-colors ease-in duration-200 hover:text-red-400"
              >
                Add to Cart
              </button>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
  import { ref, onMounted, watch } from "vue";
  import gql from "graphql-tag";

  const selectedValues = ref<{ [key: number]: string }>({});
  const configAmount = ref(0);
  const priceFormatter = new Intl.NumberFormat("en-US", {
    style: "currency",
    currency: "USD",
  });

  const selectedPrices = ref<string[]>([]);

  watch(
    selectedValues,
    (newValues) => {
      selectedPrices.value = [];
      if (product.value?.product?.components) {
        product.value.product.components.forEach((component) => {
          const selectedValue = newValues[component.id];
          if (selectedValue) {
            const variation = component.queryOptions[0].variations.nodes.find(
              (v) => v.attributes.nodes[0].value === selectedValue
            );
            if (variation) {
              selectedPrices.value.push(
                parseFloat(variation.price.replace(/[^0-9.-]+/g, "")).toFixed(2)
              );
            }
          }
        });
      }
    },
    { deep: true }
  );

  watch(
    selectedPrices,
    (newPrices) => {
      configAmount.value = newPrices.reduce((total, price) => {
        const numericPrice = parseFloat(price.replace(/[^0-9.-]+/g, ""));
        return total + (isNaN(numericPrice) ? 0 : numericPrice);
      }, 0);
    },
    { immediate: true }
  );

  const { data: product } = await useAsyncGql("getCompositeProduct", {
    id: "209",
  });

  onMounted(() => {
    if (product.value?.product?.components) {
      product.value.product.components.forEach((component) => {
        selectedValues.value[component.id] = "";
      });
    }
  });
</script>

<style scoped>
  .cursor-pointer {
    cursor: pointer;
  }

  .border-blue-500 {
    border-color: #3b82f6;
  }

  .bg-blue-100 {
    background-color: #dbeafe;
  }

  .border-gray-200 {
    border-color: #e5e7eb;
  }

  .bg-gray-100 {
    background-color: #f3f4f6;
  }

  .transition {
    transition: all 0.3s ease;
  }

  .selected-transition {
    transition: all 0.3s ease;
  }
</style>

<style scoped>
  .button-active {
    @apply bg-slate-500;
    @apply text-white;
  }
</style>
