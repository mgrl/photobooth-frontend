<template>
  <q-page-sticky position="bottom" id="gallery-toolbar" class="toolbar" v-if="item">
    <div class="q-mb-lg action-buttons col">
      <div class="q-pa-md row">
        <q-btn
          v-if="showDelete"
          stack
          rounded
          class="q-mr-sm action-button action-button-delete col-auto glass-effect"
          color="primary"
          no-caps
          icon="sym_o_delete"
          :label="$t('BTN_LABEL_GALLERY_DELETE')"
          @click="confirmDeleteDialog = true"
        />

        <q-btn
          v-if="showFilter && enableFilter"
          stack
          no-caps
          rounded
          color="primary"
          class="q-mr-sm action-button action-button-filter col-auto glass-effect"
          icon="sym_o_filter"
          :label="$t('BTN_LABEL_GALLERY_FILTER')"
          @click="invokeToggleDisplayFilter"
        />
        <q-btn
          v-if="showDownload"
          stack
          no-caps
          rounded
          color="primary"
          class="q-mr-sm action-button action-button-download col-auto glass-effect"
          icon="sym_o_download"
          :label="$t('BTN_LABEL_GALLERY_DOWNLOAD')"
          @click="openURL(`/media/full/${item.id}`)"
        />
        <ShareTriggerButtons
          v-if="showShare"
          :triggers="shareButtons"
          :direct-access-number-of-buttons="shareDirectAccessButtons"
          :current-item-is-image="['image', 'collage', 'collageimage', 'animationimage'].includes(item.media_type)"
          @trigger-action="invokeShareAction"
        ></ShareTriggerButtons>
      </div>

      <div class="q-mr-sm row flex flex-center">
        <q-badge color="grey-8" class="q-mr-xs"> <q-icon name="sym_o_image" color="white" class="q-mr-xs" /> {{ item.id }}</q-badge>
        <q-badge color="grey-8" class="q-mr-xs" v-if="props.image_number && props.images_total">
          <q-icon name="sym_o_tag" color="white" class="q-mr-xs" />
          {{
            $t('LABEL_ELEMENT_X_OF_Y', {
              no: props.image_number,
              total: props.images_total,
            })
          }}
        </q-badge>
      </div>
    </div>
  </q-page-sticky>
  <q-dialog v-model="confirmDeleteDialog">
    <q-card id="gallery-confirm-delete-dialog" class="q-pa-sm" style="min-width: 350px">
      <q-card-section class="row items-center" style="flex-wrap: nowrap">
        <q-avatar icon="sym_o_delete" color="primary" text-color="white" />
        <span class="q-ml-sm">{{ $t('MSG_CONFIRM_DELETE_IMAGE') }}</span>
      </q-card-section>

      <q-card-actions align="right">
        <q-btn v-close-popup flat :label="$t('BTN_LABEL_CANCEL')" />
        <q-btn v-close-popup :label="$t('BTN_LABEL_DELETE_IMAGE')" color="primary" @click="[invokeDeleteMediaitem(item.id), $emit('closeEvent')]" />
      </q-card-actions>
    </q-card>
  </q-dialog>
</template>

<script setup lang="ts">
import { ref } from 'vue'
import { openURL } from 'quasar'
import type { components } from 'src/dto/api'
import { default as ShareTriggerButtons, type ShareSchema } from '../ShareTriggerButtons.vue'
const confirmDeleteDialog = ref(false)

const props = defineProps<{
  item: components['schemas']['MediaitemPublic']
  image_number?: number
  images_total?: number
  showDownload: boolean
  showDelete: boolean
  showFilter: components['schemas']['GroupUiSettings']['gallery_show_filter']
  showShare: boolean
  shareDirectAccessButtons: number
  shareButtons: ShareSchema[]
  enableFilter: boolean
}>()

const emit = defineEmits<{
  triggerShareAction: [config_index: number]
  triggerToggleDisplayFilter: []
  triggerDeleteMediaitem: [id: string]
  closeEvent: [] // from quasar
}>()

function invokeShareAction(config_index: number) {
  emit('triggerShareAction', config_index)
}

function invokeToggleDisplayFilter() {
  emit('triggerToggleDisplayFilter')
}

function invokeDeleteMediaitem(id: string) {
  emit('triggerDeleteMediaitem', id)
}
</script>

<style lang="sass"></style>
