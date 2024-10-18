<script lang="ts" setup>
  import { StockStatusEnum, type AddToCartInput } from "#woo";

  const productId = ref(209); // Use your actual product ID

  const { storeSettings } = useAppConfig();
  const { addToCart, isUpdatingCart } = useCart();
  const { t } = useI18n();

  // Fetch the composite product using the product ID
  const { data } = await useAsyncGql("getCompositeProduct", {
    id: productId.value,
  });
  if (!data.value?.product) {
    throw showError({
      statusCode: 404,
      statusMessage: t("messages.shop.productNotFound"),
    });
  }

  // Extract the composite product data
  const product = ref(data.value.product);
  const quantity = ref<number>(1);

  // Initialize selected components with reactive properties
  const selectedComponents = reactive(
    product.value.components.map(() => ({
      selectedVariationId: null as string | null,
    }))
  );

  // Computed property to get the component products
  const componentProducts = computed(() => {
    return product.value.components.map((component) => {
      // Assume that the product is the first item in queryOptions
      return component.queryOptions[0];
    });
  });

  const selectProductInput = computed(() => ({
    productId: product.value.databaseId,
    quantity: quantity.value,
    components: selectedComponents.map((component, index) => ({
      componentId: product.value.components[index].id,
      selectedVariationId: component.selectedVariationId,
    })),
  })) as ComputedRef<AddToCartInput>;

  const stockStatus = computed(
    () => product.value.stockStatus || StockStatusEnum.OUT_OF_STOCK
  );

  const disabledAddToCart = computed(() => {
    return (
      !product.value ||
      stockStatus.value === StockStatusEnum.OUT_OF_STOCK ||
      isUpdatingCart.value ||
      !areAllComponentsSelected()
    );
  });

  // Function to check if all components have been selected
  function areAllComponentsSelected() {
    return selectedComponents.every((component, index) => {
      const componentProduct = componentProducts.value[index];
      if (componentProduct && componentProduct.type === "VARIABLE") {
        // Ensure a variation is selected
        return !!component.selectedVariationId;
      }
      // For simple products, no selection is needed
      return true;
    });
  }

  // When a variation is selected
  function onVariationSelected(componentIndex: number, variationId: string) {
    const component = selectedComponents[componentIndex];
    component.selectedVariationId = variationId;
  }

  function getVariationName(variation: any) {
    // If variation.name is meaningful, use it
    if (variation.name && variation.name !== "") {
      return variation.name;
    }
    // Otherwise, construct a name from attributes
    const attributeValues = variation.attributes.nodes.map(
      (attr: any) => attr.value
    );
    return attributeValues.join(" / ");
  }
</script>
<template>
  <main class="container relative py-6">
    <div v-if="product">
      <!-- Your existing components like SEOHead, Breadcrumb, etc. -->

      <div
        class="flex flex-col gap-10 md:flex-row md:justify-between lg:gap-24"
      >
        <!-- Product Image Gallery -->
        <!-- ... -->

        <!-- Product Details -->
        <div class="lg:max-w-md xl:max-w-lg md:py-2 w-full">
          <!-- Product Title and Price -->
          <!-- ... -->

          <!-- Product Form -->
          <form @submit.prevent="addToCart(selectProductInput)">
            <!-- Loop through each component -->
            <div
              v-for="(component, componentIndex) in product.components"
              :key="component.id"
              class="component mt-4 mb-8"
            >
              <h3 class="text-lg font-semibold">{{ component.title }}</h3>
              <p v-html="component.description"></p>

              <!-- Get the component product -->
              <div v-if="componentProducts[componentIndex]">
                <!-- Check if product is variable -->
                <div
                  v-if="componentProducts[componentIndex].type === 'VARIABLE'"
                >
                  <!-- Display variations as buttons -->
                  <div class="variations mt-4">
                    <div class="flex flex-wrap gap-2">
                      <button
                        v-for="variation in componentProducts[componentIndex]
                          .variations.nodes"
                        :key="variation.id"
                        @click.prevent="
                          onVariationSelected(componentIndex, variation.id)
                        "
                        :class="[
                          'px-4 py-2 rounded',
                          selectedComponents[componentIndex]
                            .selectedVariationId === variation.id
                            ? 'bg-blue-500 text-white'
                            : 'bg-gray-200',
                        ]"
                      >
                        {{ getVariationName(variation) }} -
                        {{ variation.price || "N/A" }}
                      </button>
                    </div>
                  </div>
                </div>
                <!-- Simple Product -->
                <div v-else class="mt-4">
                  <!-- Display any additional information for simple products if needed -->
                  <p>This is a simple product with no options.</p>
                </div>
              </div>
            </div>

            <!-- Add to Cart Button -->
            <div
              class="fixed bottom-0 left-0 z-10 flex items-center w-full gap-4 p-4 mt-12 bg-white md:static md:bg-transparent bg-opacity-90 md:p-0"
            >
              <input
                v-model="quantity"
                type="number"
                min="1"
                aria-label="Quantity"
                class="bg-white border rounded-lg flex text-left p-2.5 w-20 gap-4 items-center justify-center focus:outline-none"
              />
              <AddToCartButton
                class="flex-1 w-full md:max-w-xs"
                :disabled="disabledAddToCart"
                :class="{ loading: isUpdatingCart }"
              />
            </div>
          </form>

          <!-- Additional Components like Wishlist, Share, etc. -->
          <!-- ... -->
        </div>
      </div>

      <!-- Product Tabs, Related Products, etc. -->
      <!-- ... -->
    </div>
  </main>
</template>
