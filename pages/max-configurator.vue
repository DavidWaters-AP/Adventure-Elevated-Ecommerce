<script lang="ts" setup>
  import { StockStatusEnum, ProductTypesEnum, type AddToCartInput } from "#woo";
  import { watch, onMounted } from "vue";

  const route = useRoute();
  const { storeSettings } = useAppConfig();
  const { arraysEqual, formatArray, checkForVariationTypeOfAny } = useHelpers();
  const { addToCart, isUpdatingCart } = useCart();
  const { t } = useI18n();

  // Fetch the composite product using the product ID
  const productId = ref("209"); // Replace with actual product ID
  const { data } = await useAsyncGql("getCompositeProduct", {
    id: productId.value,
  });

  if (!data.value?.product) {
    throw showError({
      statusCode: 404,
      statusMessage: t("messages.shop.productNotFound"),
    });
  }

  const product = ref(data.value.product);
  const quantity = ref<number>(1);

  // Initialize selected components with reactive properties
  const selectedComponents = ref(
    product.value.components.map(() => ({
      selectedAttributes: {} as Record<string, string>,
      selectedVariationId: null as string | null,
    }))
  );

  // Computed property to get the component products
  const componentProducts = computed(() => {
    return product.value.components.map((component) => {
      return component.queryOptions[0];
    });
  });

  // Prepare attribute options with prices
  const componentAttributes = computed(() => {
    return componentProducts.value.map((product) => {
      if (product && product.type === "VARIABLE") {
        const attributes = product.attributes.nodes;
        const variations = product.variations.nodes;

        const attributeOptions = attributes.map((attribute) => {
          const optionsWithPrices = attribute.options.map((optionValue) => {
            const matchingVariations = variations.filter((variation) =>
              variation.attributes.nodes.some(
                (attr) =>
                  attr.name.toLowerCase() === attribute.name.toLowerCase() &&
                  attr.value === optionValue
              )
            );

            const prices = matchingVariations
              .map((v) => {
                if (v.price) {
                  const priceStr = v.price.replace(/[^0-9.-]+/g, "");
                  return parseFloat(priceStr);
                }
                return null;
              })
              .filter((price) => price !== null);

            const minPrice = prices.length > 0 ? Math.min(...prices) : null;

            return {
              optionValue,
              price: minPrice,
            };
          });

          return {
            attributeName: attribute.name,
            attributeLabel: attribute.label,
            options: optionsWithPrices,
          };
        });
        return attributeOptions;
      }
      return [];
    });
  });

  // Computed property for stock status
  const stockStatus = computed(() => {
    if (product.value.type === ProductTypesEnum.VARIABLE) {
      return product.value.stockStatus || StockStatusEnum.OUT_OF_STOCK;
    }
    return product.value.stockStatus || StockStatusEnum.OUT_OF_STOCK;
  });

  // Computed property for disabled Add to Cart button
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
    return selectedComponents.value.every((component, index) => {
      const componentProduct = componentProducts.value[index];
      if (componentProduct && componentProduct.type === "VARIABLE") {
        const requiredAttributes = componentProduct.attributes.nodes.map(
          (attr) => attr.name.toLowerCase()
        );
        const selectedAttributes = Object.keys(component.selectedAttributes);
        const allAttributesSelected = requiredAttributes.every((attr) =>
          selectedAttributes.includes(attr)
        );
        return allAttributesSelected && !!component.selectedVariationId;
      }
      return true; // For simple products, no selection is needed
    });
  }

  // Function to update selected attributes
  function onAttributeOptionSelected(
    componentIndex: number,
    attributeName: string,
    optionValue: string
  ) {
    const component = selectedComponents.value[componentIndex];
    const normalizedAttributeName = attributeName.toLowerCase();
    component.selectedAttributes[normalizedAttributeName] = optionValue;
    component.selectedVariationId = null; // Reset variation when attributes change

    // Find and set the matching variation
    const componentProduct = componentProducts.value[componentIndex];
    if (componentProduct && componentProduct.type === "VARIABLE") {
      const matchingVariation = findMatchingVariation(
        componentProduct,
        component.selectedAttributes
      );
      if (matchingVariation) {
        component.selectedVariationId = matchingVariation.databaseId;
      } else {
        component.selectedVariationId = null;
      }
    }
  }

  // Function to find matching variation based on selected attributes
  function findMatchingVariation(
    product: any,
    selectedAttributes: Record<string, string>
  ) {
    if (!product.variations || !product.variations.nodes) return null;
    const selectedAttributesEntries = Object.entries(selectedAttributes);

    return product.variations.nodes.find((variation: any) => {
      const variationAttributes = variation.attributes.nodes.reduce(
        (acc: Record<string, string>, attr: any) => {
          const normalizedAttrName = attr.name.toLowerCase();
          acc[normalizedAttrName] = attr.value;
          return acc;
        },
        {}
      );

      return selectedAttributesEntries.every(
        ([key, value]) => variationAttributes[key] === value
      );
    });
  }

  // Computed property for selectProductInput
  const selectProductInput = computed<AddToCartInput>(() => ({
    productId: product.value.databaseId,
    quantity: quantity.value,
    components: selectedComponents.value.map((component, index) => ({
      componentId: product.value.components[index].id,
      selectedVariationId: component.selectedVariationId,
      selectedAttributes: component.selectedAttributes,
    })),
  }));

  // Lifecycle hooks
  onMounted(async () => {
    try {
      const { product } = await GqlGetStockStatusComposite({
        id: productId.value,
      });
      if (product) mergeLiveStockStatus(product as Product);
    } catch (error: any) {
      const errorMessage = error?.gqlErrors?.[0].message;
      if (errorMessage) console.error(errorMessage);
    }
  });

  watch(route, async (newRoute) => {
    productId.value = newRoute.params.slug as string;
    const { data } = await useAsyncGql("getCompositeProduct", {
      id: productId.value,
    });
    product.value = data.value.product;
    selectedComponents.value = product.value.components.map(() => ({
      selectedAttributes: {},
      selectedVariationId: null,
    }));
  });
</script>

<template>
  <main class="container relative py-6">
    <div v-if="product">
      <!-- SEO Head -->
      <SEOHead :info="product" />

      <!-- Breadcrumb -->
      <Breadcrumb
        :product
        class="mb-6"
        v-if="storeSettings.showBreadcrumbOnSingleProduct"
      />

      <div
        class="flex flex-col gap-10 md:flex-row md:justify-between lg:gap-24"
      >
        <!-- Product Image Gallery 
        <ProductImageGallery
          v-if="product.image"
          class="relative flex-1"
          :main-image="product.image.sourceUrl"
          :gallery="product.galleryImages.nodes"
          :node="product"
        />
        
        <NuxtImg
          v-else
          class="relative flex-1 skeleton"
          src="/images/placeholder.jpg"
          :alt="product?.name || 'Product'"
        />
        -->
        <div class="lg:max-w-md xl:max-w-lg md:py-2 w-full">
          <div class="flex justify-between mb-4">
            <div class="flex-1">
              <h1
                class="flex flex-wrap items-center gap-2 mb-2 text-2xl font-sesmibold"
              >
                {{ product.name }}
                <LazyWPAdminLink
                  :link="`/wp-admin/post.php?post=${product.databaseId}&action=edit`"
                >
                  Edit
                </LazyWPAdminLink>
              </h1>
              <StarRating
                :rating="product.averageRating || 0"
                :count="product.reviewCount || 0"
                v-if="storeSettings.showReviews"
              />
            </div>
            <ProductPrice
              class="text-xl"
              :sale-price="product.salePrice"
              :regular-price="product.regularPrice"
            />
          </div>

          <!-- Product Details -->
          <div class="grid gap-2 my-8 text-sm empty:hidden">
            <div v-if="!isExternalProduct" class="flex items-center gap-2">
              <span class="text-gray-400"
                >{{ $t("messages.shop.availability") }}:</span
              >
              <StockStatus :stockStatus @updated="mergeLiveStockStatus" />
            </div>
            <div
              class="flex items-center gap-2"
              v-if="storeSettings.showSKU && product.sku"
            >
              <span class="text-gray-400">{{ $t("messages.shop.sku") }}:</span>
              <span>{{ product.sku || "N/A" }}</span>
            </div>
          </div>

          <div class="mb-8 font-light prose" v-html="product.description"></div>

          <hr />

          <form @submit.prevent="addToCart(selectProductInput)">
            <!-- Loop through each component -->
            <div
              v-for="(component, componentIndex) in product.components"
              :key="component.id"
              class="component mt-4 mb-8"
            >
              <h3 class="text-lg font-semibold">
                {{ component.title }} -
                {{ componentProducts[componentIndex]?.name }}
              </h3>
              <p v-html="component.description"></p>

              <!-- Get the component product -->
              <div v-if="componentProducts[componentIndex]">
                <!-- Check if product is variable -->
                <div
                  v-if="componentProducts[componentIndex]?.type === 'VARIABLE'"
                >
                  <!-- Display attributes and options -->
                  <div
                    v-for="attribute in componentAttributes[componentIndex]"
                    :key="attribute.attributeName"
                    class="mt-4"
                  >
                    <h4 class="text-md font-semibold">
                      {{ attribute.attributeLabel }}
                    </h4>
                    <div class="flex flex-wrap gap-2 mt-2">
                      <button
                        v-for="option in attribute.options"
                        :key="option.optionValue"
                        @click.prevent="
                          onAttributeOptionSelected(
                            componentIndex,
                            attribute.attributeName,
                            option.optionValue
                          )
                        "
                        :class="[
                          'px-4 py-2 rounded border',
                          selectedComponents.value[componentIndex]
                            ?.selectedAttributes[
                            attribute.attributeName.toLowerCase()
                          ] === option.optionValue
                            ? 'bg-blue-500 text-white border-blue-500'
                            : 'bg-gray-100 border-gray-300',
                        ]"
                      >
                        {{ option.optionValue }} -
                        {{
                          option.price !== null
                            ? "$" + option.price.toFixed(2)
                            : "N/A"
                        }}
                      </button>
                    </div>
                  </div>
                </div>
                <!-- Simple Product -->
                <div v-else class="mt-4">
                  <div
                    v-for="option in componentProducts[componentIndex]
                      .queryOptions"
                    :key="option.id"
                  >
                    <button
                      @click.prevent="
                        onSimpleProductSelected(
                          componentIndex,
                          option.id,
                          option.name
                        )
                      "
                      :class="[
                        'px-4 py-2 rounded border',
                        selectedComponents.value[componentIndex]
                          ?.selectedVariationId === option.id
                          ? 'bg-blue-500 text-white border-blue-500'
                          : 'bg-gray-100 border-gray-300',
                      ]"
                    >
                      {{ option.name }} - {{ option.price }}
                    </button>
                  </div>
                </div>
              </div>
            </div>

            <!-- Add to Cart Button -->
            <div
              v-if="isVariableProduct || isSimpleProduct"
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

          <!-- Product Categories -->
          <div
            v-if="
              storeSettings.showProductCategoriesOnSingleProduct &&
              product.productCategories.nodes
            "
            class="grid gap-2 my-8 text-sm"
          >
            <div class="flex items-center gap-2">
              <span class="text-gray-400"
                >{{ $t("messages.shop.category", 2) }}:</span
              >
              <div class="product-categories">
                <NuxtLink
                  v-for="category in product.productCategories.nodes"
                  :key="category.slug"
                  :to="`/product-category/${decodeURIComponent(category.slug)}`"
                  class="hover:text-primary"
                  :title="category.name"
                >
                  {{ category.name }}<span class="comma">, </span>
                </NuxtLink>
              </div>
            </div>
          </div>

          <hr />

          <!-- Wishlist and Share Buttons -->
          <div class="flex flex-wrap gap-4">
            <WishlistButton :product="product" />
            <ShareButton :product="product" />
          </div>
        </div>
      </div>

      <!-- Product Tabs -->
      <div v-if="product.description || product.reviews" class="my-32">
        <ProductTabs :product="product" />
      </div>

      <!-- Related Products -->
      <div
        v-if="product.related && storeSettings.showRelatedProducts"
        class="my-32"
      >
        <div class="mb-4 text-xl font-semibold">
          {{ $t("messages.shop.youMayLike") }}
        </div>
        <ProductRow
          :products="product.related.nodes"
          class="grid-cols-2 md:grid-cols-4 lg:grid-cols-5"
        />
      </div>
    </div>
  </main>
</template>

<style scoped>
  .product-categories > a:last-child .comma {
    display: none;
  }

  input[type="number"]::-webkit-inner-spin-button {
    opacity: 1;
  }
</style>
