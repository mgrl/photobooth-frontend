<template>
  <div class="row q-gutter-md action-buttons">
    <div v-for="(trigger, index) in triggers" :key="index">
      <q-btn
        v-if="trigger.show_button && !(isEmpty(trigger.title) && isEmpty(trigger.icon))"
        stack
        :color="trigger.use_custom_color ? 'custom' : 'primary'"
        no-caps
        rounded
        class="action-button col-auto glass-effect"
        :class="['action-button-' + index]"
        :style="{ '--bg-custom': trigger.use_custom_color ? trigger.custom_color : null }"
        @click="triggerButtonClick(trigger.action, trigger.config_index)"
      >
        <q-icon v-if="!isEmpty(trigger.icon)" :name="`sym_o_${trigger.icon}`" />
        <div style="white-space: nowrap" class="gt-sm">{{ trigger.title }}</div>
      </q-btn>
    </div>
  </div>
</template>

<script setup lang="ts">
import { isEmpty } from 'lodash'

defineProps<{
  triggers: TriggerSchema[]
}>()

export interface TriggerSchema {
  show_button: boolean
  title: string
  icon?: string // icon is optional
  action: string
  config_index: number
  use_custom_color: boolean
  custom_color: string
}

const emit = defineEmits<{
  triggerAction: [action: string, config_index: number]
}>()

function triggerButtonClick(action: string, config_index: number) {
  // console.log(`trigger action ${action}, config_index ${config_index}`);
  emit('triggerAction', action, config_index)
}
</script>
