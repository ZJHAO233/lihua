<template>
  <div class="login-setting scrollbar">
    <a-carousel style="width: 600px;border: none; border-radius: 8px;" :dots="false" ref="carouselRef">
      <component :is="component"
                 :key="index"
                 v-for="(component, index) in componentList"
                 @back="handleBack"
                 @skip="handleSkip"
                 @next="handleNext"
                 @goLogin="emits('goLogin')"
      />
    </a-carousel>
  </div>

</template>

<script setup lang="ts">
import {ref, useTemplateRef} from "vue";
import type {CarouselRef} from "ant-design-vue/es/carousel";

const carouselRef = useTemplateRef<CarouselRef>("carouselRef")
const emits = defineEmits(['goLogin'])
// 需要加载的设置项集合
const componentList = ref<string[]>(
    [
      'LoginSettingStart',
      'LoginSettingComplete'
    ]
)
// 接收需要加载的配置项
const props = defineProps<{
  componentNames: string[];
}>()

// 组合配置项
componentList.value.splice(1, 0, ...props.componentNames)

// 上一页
const handleBack = () => {
  carouselRef.value?.prev()
}
// 跳过
const handleSkip = (loading:boolean) => {
  if (!loading) {
    carouselRef.value?.next()
  }
}
// 下一页
const handleNext = (loading:boolean) => {
  if (!loading) {
    carouselRef.value?.next()
  }
}
</script>

<style scoped>
.login-setting{
  margin: auto;
  max-height: 100vh;
}
:deep(.slick-list) {
  border-radius: 8px;
}
</style>
