<template>
  <q-layout view="hhh Lpr ffr" @click="headercountdowntimer = false">
    <div v-if="currentMediaitem" class="fixed-full">
      <!-- v-if above is guard to hide all content that would fail if no currentMediaitem is avail -->
      <q-header class="bg-primary text-white">
        <HeaderCountdownTimer
          v-if="headercountdowntimer"
          :duration="configurationStore.configuration.uisettings.AUTOCLOSE_NEW_ITEM_ARRIVED"
          @trigger-timeout="$router.push({ path: '/' })"
        ></HeaderCountdownTimer>
        <HeaderProcessing v-if="displayIndeterminateProgressbar"></HeaderProcessing>
      </q-header>

      <q-drawer v-if="showFilter" id="gallery-drawer-filters" v-model="rightDrawerOpen" class="q-pa-sm" side="right" overlay elevated>
        <DrawerFilter
          v-if="rightDrawerOpen"
          :id="currentMediaitem.id"
          :available-filter="getAvailableFilters"
          @trigger-apply-filter="doApplyFilter"
        ></DrawerFilter>
      </q-drawer>

      <q-page-container class="q-pa-none galleryimagedetail full-height">
        <q-page class="full-height">
          <PageCarouselView
            :mediaitem-id="currentMediaitem.id"
            :sliced-images="mediacollectionStore.collection"
            @trigger-changed-item="onCarouselTransition"
            @click="rightDrawerOpen = false"
          />

          <q-page-sticky position="top-right" class="q-ma-lg" v-if="configurationStore.configuration.uisettings.gallery_show_qrcode">
            <PageQrCode
              :url="qrShareUrl"
              :text-above="configurationStore.configuration.uisettings.qrcode_text_above"
              :text-below="configurationStore.configuration.uisettings.qrcode_text_below"
            />
          </q-page-sticky>

          <PageToolbar
            :item="currentMediaitem"
            :show-filter="configurationStore.configuration.uisettings.gallery_show_filter"
            :enable-filter="getFilterAvailable(currentMediaitem.media_type)"
            :show-share="
              configurationStore.configuration.uisettings.gallery_show_shareprint && configurationStore.configuration.share.sharing_enabled
            "
            :share-direct-access-buttons="configurationStore.configuration.share.number_direct_access_buttons"
            :share-buttons="shareButtons"
            :show-delete="configurationStore.configuration.uisettings.gallery_show_delete"
            :show-download="configurationStore.configuration.uisettings.gallery_show_download"
            :image_number="currentMediaitemNumber"
            :images_total="mediacollectionStore.collection_number_of_items"
            @trigger-toggle-display-filter="rightDrawerOpen = !rightDrawerOpen"
            @trigger-delete-mediaitem="doDeleteItem"
            @trigger-share-action="doShareAction"
          ></PageToolbar>
        </q-page>
      </q-page-container>
    </div>
    <div v-else>
      <!-- usually not visible but if page is called with wrong id or the page was not closed after an item is deleted -->
      <ItemNotAvailableError />
    </div>

    <ReturnButton @trigger-return="$router.back()"></ReturnButton>
    <!-- go back to index after inactivity -->
    <RouteAfterTimeout route="/" :timeout-ms="configurationStore.configuration.uisettings.show_frontpage_timeout * 60 * 1000"></RouteAfterTimeout>
  </q-layout>
</template>

<script setup lang="ts">
import { useConfigurationStore } from '../stores/configuration-store'
import { useMediacollectionStore } from '../stores/mediacollection-store'
import { ref, onBeforeMount, computed, onMounted, watch } from 'vue'
import { default as PageToolbar } from '../components/mediaviewer/PageToolbar.vue'
import { default as HeaderCountdownTimer } from '../components/mediaviewer/HeaderCountdownTimer.vue'
import { default as HeaderProcessing } from '../components/mediaviewer/HeaderProcessing.vue'
import { default as DrawerFilter } from '../components/mediaviewer/DrawerFilter.vue'
import { default as PageQrCode } from '../components/mediaviewer/PageQrCode.vue'
import { default as PageCarouselView } from '../components/mediaviewer/PageCarouselView.vue'
import ItemNotAvailableError from '../components/ItemNotAvailableError.vue'
import { useRouter, useRoute } from 'vue-router'
import { type ShareSchema } from '../components/ShareTriggerButtons.vue'
import { remoteProcedureCall } from '../util/fetch_api.js'
import { default as ReturnButton } from '../components/ReturnButton.vue'
import RouteAfterTimeout from 'src/components/RouteAfterTimeout.vue'

const route = useRoute()
const router = useRouter()
const configurationStore = useConfigurationStore()
const mediacollectionStore = useMediacollectionStore()
const selectedMediaitemId = ref('')
const rightDrawerOpen = ref(false)
const headercountdowntimer = ref(false) // likely not used here, move to newitempresenter and approval...
const displayIndeterminateProgressbar = ref(false)

const props = defineProps({
  startTimer: Boolean,
})

onBeforeMount(() => {
  selectedMediaitemId.value = route.params.id as string
})
watch(route, (to) => {
  selectedMediaitemId.value = to.params.id as string
})
onMounted(() => {
  headercountdowntimer.value = props.startTimer
})
const onCarouselTransition = (newMediaitemId: string) => {
  selectedMediaitemId.value = newMediaitemId
}

const currentMediaitem = computed(() => {
  return getMediaitemById(selectedMediaitemId.value)
})

const qrShareUrl = computed(() => {
  if (configurationStore.configuration.qrshare.enabled) {
    const qrShareServiceUrl = configurationStore.configuration.qrshare.shareservice_url
    return `${qrShareServiceUrl}?action=download&id=${selectedMediaitemId.value}`
  } else {
    const customUrl = configurationStore.configuration.qrshare.share_custom_qr_url

    // get filename from processed/unprocessed path. Could be improved in future, when the backend delivers the actual filename
    const getFileName = (filepath: string) => filepath.split(/[\\/]/).pop()

    return customUrl.replace('{filename}', getFileName(currentMediaitem.value.processed)).replace('{identifier}', selectedMediaitemId.value)
  }
})

const currentMediaitemNumber = computed(() => {
  return mediacollectionStore.collection.findIndex((mediaitem) => mediaitem.id == selectedMediaitemId.value) + 1
})
const showFilter = computed(() => {
  return configurationStore.configuration.uisettings.gallery_show_filter && getFilterAvailable(currentMediaitem.value.media_type)
})
const shareButtons = computed(() => {
  const result: ShareSchema[] = []

  const share_config = configurationStore.configuration.share.actions
  share_config.forEach((action, index: number) => {
    const trigger: ShareSchema = {
      config_index: index,
      handles_images_only: action.handles_images_only,
      show_button: action.trigger.ui_trigger.show_button,
      title: action.trigger.ui_trigger.title,
      icon: action.trigger.ui_trigger.icon,
    }

    result.push(trigger)
  })

  return result
})
const getMediaitemById = (id: string) => {
  return mediacollectionStore.collection.find((mediaitem) => mediaitem.id == id)
}

const getFilterAvailable = (media_type: string) => {
  return ['image', 'collageimage', 'animationimage'].includes(media_type)
}

const getAvailableFilters = computed(() => {
  if( configurationStore.configuration.mediaprocessing.filtertype == 'pilgram2') {
      return configurationStore.configuration.uisettings.gallery_filter_userselectable
  }
  else if(configurationStore.configuration.mediaprocessing.filtertype == 'stablediffusion') {
      return configurationStore.configuration.uisettings.gallery_filter_stablediffusion_userselectable
  }
  return []
})

const doApplyFilter = (id: string, filter: string) => {
  displayIndeterminateProgressbar.value = true
  fetch(`/api/mediaprocessing/applyfilter/${id}/${filter}`)
    .then((response) => {
      if (!response.ok) {
        throw new Error('Server returned ' + response.status)
      }

      displayIndeterminateProgressbar.value = false
    })
    .catch((err) => {
      console.error(err)
      displayIndeterminateProgressbar.value = false
    })
}
const doDeleteItem = (id: string) => {
  selectedMediaitemId.value = undefined
  mediacollectionStore.deleteItem(id)
  router.back()
}
const doShareAction = (config_index: number) => {
  console.log(selectedMediaitemId.value, config_index)
  remoteProcedureCall(`/api/share/actions/${selectedMediaitemId.value}/${config_index}`)
}
</script>
