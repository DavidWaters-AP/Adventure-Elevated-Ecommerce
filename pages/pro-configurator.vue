<script lang="ts" setup>
  import { StockStatusEnum, type AddToCartInput } from "#woo";

  const route = useRoute();
  const { storeSettings } = useAppConfig();
  const { addToCart, isUpdatingCart } = useCart();
  const { t } = useI18n();
  const productId = ref("209"); // Use the provided post ID

  // Fetch the composite product using the product ID
  const { data } = await useAsyncGql("getCompositeProduct", {
    id: "209",
  });
  if (!data.value?.product) {
    throw showError({
      statusCode: 404,
      statusMessage: t("messages.shop.productNotFound"),
    });
  }
</script>

<template>
  <div>empty</div>
</template>
