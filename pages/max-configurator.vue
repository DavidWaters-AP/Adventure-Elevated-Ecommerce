<script setup lang="ts">
  import { ref, computed } from "vue";

  const productId = ref(209);

  const { data } = await useAsyncGql("getCompositeProduct", {
    id: productId.value,
  });
  if (!data.value?.product) {
    throw showError({
      statusCode: 404,
      statusMessage: t("messages.shop.productNotFound"),
    });
  }

  // Assuming you have a GraphQL query that fetches the composite product data
  const product = ref(data);

  // Component products
  const componentProducts = computed(() => {
    return product.value.components.map(
      (component) => component.queryOptions[0]
    );
  });

  // Selected variations for each component
  const selectedVariations = ref([]);

  // Quantity
  const quantity = ref(1);

  // Function to handle attribute changes
  function onAttributeChange(componentIndex, attributeName, optionValue) {
    const component = componentProducts.value[componentIndex];

    // Find the matching variation
    const matchingVariation = component.variations.nodes.find((v) =>
      v.attributes.nodes.some(
        (a) => a.name === attributeName && a.value === optionValue
      )
    );

    // Update selected variations
    selectedVariations.value[componentIndex] = matchingVariation;

    // Update quantity if necessary
    if (matchingVariation) {
      quantity.value = matchingVariation.quantity;
    }
  }

  // Function to prepare add-to-cart data
  function prepareAddToCartData() {
    return {
      productId: product.value.databaseId,
      quantity: quantity.value,
      components: selectedVariations.value.map((variation, index) => ({
        componentId: product.value.components[index].id,
        variationId: variation.databaseId,
        attributes: variation.attributes.nodes.map((attr) => ({
          attributeName: attr.name,
          attributeValue: attr.value,
        })),
      })),
    };
  }
</script>

<template>
  <main class="container relative py-6">
    <div v-if="product">
      <h1 class="text-2xl font-bold mb-4">{{ product.name }}</h1>

      <form @submit.prevent="addToCart(prepareAddToCartData())">
        <div
          v-for="(component, index) in product.components"
          :key="component.id"
          class="mb-6 border-b pb-4"
        >
          <h2 class="text-xl font-semibold mb-2">{{ component.title }}</h2>

          <div v-if="componentProducts[index].type === 'VARIABLE'">
            <div
              v-for="attribute in componentProducts[index].attributes.nodes"
              :key="attribute.name"
              class="mb-4"
            >
              <h3 class="text-lg font-semibold mb-1">{{ attribute.label }}</h3>
              <div class="flex flex-wrap gap-2">
                <button
                  v-for="option in attribute.options"
                  :key="option"
                  @click.prevent="
                    onAttributeChange(index, attribute.name, option)
                  "
                  :class="{
                    'bg-blue-500 text-white': selectedVariations.value[
                      index
                    ]?.attributes.nodes.some(
                      (a) => a.name === attribute.name && a.value === option
                    ),
                  }"
                  class="px-4 py-2 rounded border hover:bg-gray-100 transition-colors"
                >
                  {{ option }}
                </button>
              </div>
            </div>
          </div>

          <p
            v-if="componentProducts[index].description"
            v-html="componentProducts[index].description"
            class="text-sm text-gray-600 mb-2"
          ></p>

          <div
            v-if="selectedVariations[index] && selectedVariations[index].price"
          >
            <p class="text-lg font-semibold mb-1">
              Price: {{ selectedVariations[index].price }}
            </p>
            <p class="text-sm text-gray-600">
              Stock: {{ selectedVariations[index].stockQuantity }} in stock
            </p>
          </div>
        </div>

        <div class="flex items-center justify-between mt-6">
          <div>
            <label for="quantity" class="text-sm font-semibold mr-2"
              >Quantity:</label
            >
            <input
              id="quantity"
              v-model="quantity"
              type="number"
              min="1"
              class="w-20 p-2 mr-2 border rounded"
            />
          </div>
          <button type="submit" class="bg-primary text-white px-4 py-2 rounded">
            Add to Cart
          </button>
        </div>
      </form>
    </div>
  </main>
</template>
