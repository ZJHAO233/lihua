<template>
  <a-flex align="center" justify="space-between">
    <left v-if="themeStore.layoutType === 'sider-header'"/>
    <right/>
  </a-flex>
</template>
<script setup lang="ts">
import Left from "@/layout/head/components/Left.vue"
import Right from "@/layout/head/components/Right.vue"
import { useThemeStore } from "@/stores/modules/theme";
const themeStore = useThemeStore()
</script>

