<template>
  <a-dropdown v-model:open="open" trigger="click" :getPopupContainer="(triggerNode:Document) => triggerNode.parentNode">
    <a-tooltip title="默认部门">
      <span class="header-right-item header-right" style="max-width: 130px">
        <a-typography-text ellipsis
                           class="text-default-color"
                           :content="userStore.defaultDeptName"
                           style="vertical-align: middle"/>
      </span>
    </a-tooltip>
    <template #overlay>
      <div class="default-dept-card scrollbar">
        <default-dept @dept-select="() => open = false"/>
      </div>
    </template>
  </a-dropdown>
</template>

<script setup lang="ts">
import {useUserStore} from "@/stores/modules/user.ts";
import DefaultDept from "@/components/dept-select/index.vue"
import {ref} from "vue";
const userStore = useUserStore();
const open = ref<boolean>(false)
</script>

<style scoped>
.default-dept-card {
  max-height: 500px;
  box-shadow: var(--lihua-light-box-shadow);
  padding: 16px 16px 0 16px;
  background-color: #ffffff;
  border-radius: 8px;
}
</style>
<style>
[data-theme=dark] {
  .default-dept-card {
    background-color: #1f1f1f;
  }
}
</style>

