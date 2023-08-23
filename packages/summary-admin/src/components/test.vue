<script lang="ts" setup>
import { hello } from "@summary-project/shared-test-copy";

const handleClick = () => {
  hello();
  console.log("123");
};
</script>
<template>
  <div class="test">
    <button @click="handleClick">123</button>
  </div>
</template>

<style scoped></style>
