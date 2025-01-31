<template>
  <q-page id="index-page" class="q-pa-none column full-height">
    <preview-stream
      v-if="showPreview"
      :frame-overlay-image="frameOverlayImage"
      :enable-blurred-background-stream="configurationStore.configuration.uisettings.livestream_blurredbackground"
      :enable-mirror-effect="configurationStore.configuration.uisettings.livestream_mirror_effect"
    ></preview-stream>

    <!-- layer display processing spinner grid to show user computer working hard -->
    <div v-if="showProcessing" class="full-height full-width column justify-center content-center" style="position: absolute">
      <q-spinner-grid size="20em" />
    </div>

    <!-- layer display the countdown timer -->
    <div
      v-if="showCountdownCounting"
      id="frontpage-countdown"
      class="full-height full-width column justify-center content-center"
      style="position: absolute"
    >
      <countdown-timer
        ref="countdowntimer"
        :duration="stateStore.duration"
        :message-duration="configurationStore.configuration.uisettings.TAKEPIC_MSG_TIME"
        :message-text="configurationStore.configuration.uisettings.TAKEPIC_MSG_TEXT"
      ></countdown-timer>
    </div>

    <!-- layer display the front page text -->
    <!-- eslint-disable-next-line vue/no-v-html -->
    <div v-if="showFrontpage" id="frontpage_text" v-html="configurationStore.configuration.uisettings.FRONTPAGE_TEXT"></div>

    <q-page-sticky position="bottom" class="q-mb-lg">
      <div v-if="showFrontpage">
        <FrontpageTriggerButtons :triggers="triggerButtons" @trigger-action="invokeAction"></FrontpageTriggerButtons>
      </div>
    </q-page-sticky>

    <q-page-sticky position="top-left" class="q-ma-lg">
      <div v-if="showFrontpage">
        <div class="q-gutter-md">
          <q-btn
            v-if="configurationStore.configuration.uisettings.show_gallery_on_frontpage"
            id="frontpage-button-to-gallery"
            color="primary"
            no-caps
            rounded
            to="/gallery"
            class="action-button glass-effect"
          >
            <q-icon left name="sym_o_photo_library" />
            <div class="gt-sm">{{ $t('BTN_LABEL_MAINPAGE_TO_GALLERY') }}</div>
          </q-btn>
        </div>
      </div>
    </q-page-sticky>

    <q-page-sticky position="top-right" class="q-ma-lg">
      <div v-if="showFrontpage">
        <div class="q-gutter-md">
          <q-btn
            v-if="configurationStore.configuration.uisettings.show_admin_on_frontpage"
            id="frontpage-button-to-admin"
            rounded
            color="transparent"
            no-caps
            class="action-button action-button-admin glass-effect"
            :class="{ 'action-button-admin-invisible': adminButtonInvisible }"
            @click="onBtnAdminClick"
          >
            <q-icon left name="sym_o_admin_panel_settings" />
            <div class="gt-sm">{{ $t('BTN_LABEL_MAINPAGE_TO_ADMIN') }}</div>
          </q-btn>
        </div>
      </div>
    </q-page-sticky>

    <!-- video state controls -->
    <q-page-sticky v-if="showRecording" id="frontpage-indicator-recording" position="top" :offset="[0, 25]" align="center">
      <q-spinner-puff color="red" size="10em" />
      <br />
      <q-btn flat color="red" label="Stop recording" @click="stopRecordingVideo()" />
    </q-page-sticky>

    <!-- auto-start slideshow after timeout -->
    <RouteAfterTimeout
      v-if="configurationStore.configuration.uisettings.enable_automatic_slideshow && mediacollectionStore.collection_number_of_items > 0"
      route="/slideshow/random"
      :timeout-ms="configurationStore.configuration.uisettings.show_automatic_slideshow_timeout * 1000"
      :warning-message="$t('MSG_WARN_BEFORE_AUTO_SLIDESHOW')"
    ></RouteAfterTimeout>
  </q-page>
</template>

<script setup lang="ts">
import { useRouter } from 'vue-router'
import { watchDebounced } from '@vueuse/core'
import { computed, ref } from 'vue'
import { remoteProcedureCall } from '../util/fetch_api.js'
import { useStateStore } from '../stores/state-store'
import { useConfigurationStore } from '../stores/configuration-store'
import CountdownTimer from '../components/CountdownTimer.vue'
import type { TriggerSchema } from '../components/FrontpageTriggerButtons.vue'
import { default as FrontpageTriggerButtons } from '../components/FrontpageTriggerButtons.vue'
import { default as PreviewStream } from '../components/PreviewStream.vue'
import _ from 'lodash'
import RouteAfterTimeout from 'src/components/RouteAfterTimeout.vue'
import { useMediacollectionStore } from '../stores/mediacollection-store'

const mediacollectionStore = useMediacollectionStore()
const stateStore = useStateStore()
const configurationStore = useConfigurationStore()
const router = useRouter()
const btnAdminClickCounter = ref(0)

watchDebounced(
  btnAdminClickCounter,
  () => {
    if (btnAdminClickCounter.value >= 5) {
      router.push('/admin')
    }
    btnAdminClickCounter.value = 0
  },
  { debounce: 500 },
)
const triggerButtons = computed(() => {
  const result: TriggerSchema[] = []

  Object.entries(configurationStore.configuration.actions).forEach(([key, actions]) => {
    actions.forEach((action, index: number) => {
      const trigger: TriggerSchema = {
        action: `actions/${key}`,
        config_index: index,
        show_button: action.trigger.ui_trigger.show_button,
        title: action.trigger.ui_trigger.title,
        icon: action.trigger.ui_trigger.icon,
        use_custom_color: action.trigger.ui_trigger.use_custom_color,
        custom_color: action.trigger.ui_trigger.custom_color,
      }

      result.push(trigger)
    })
  })

  console.log(result)

  return result
})

const showProcessing = computed(() => {
  const capture = stateStore.state == 'capture'
  const capturesCompleted = stateStore.state == 'captures_completed'

  return capturesCompleted || (capture && !showCountdownCounting.value)
})

const showRecording = computed(() => {
  return stateStore.state == 'record'
})

const adminButtonInvisible = computed(() => {
  return configurationStore.configuration.uisettings.admin_button_invisible
})
const showCountdownCounting = computed(() => {
  const machineCounting = stateStore.state == 'counting'
  const capture = stateStore.state == 'capture'

  return (stateStore.duration && stateStore.duration > 0 && machineCounting) || capture
})

const frameOverlayImage = computed(() => {
  if (showCountdownCounting.value) {
    const enable_action_frame_overlay = _.get(stateStore.configuration_set, 'processing.img_frame_enable', false)
    const action_frame_overlay_image = _.get(stateStore.configuration_set, 'processing.img_frame_file', '')
    if (enable_action_frame_overlay) {
      return action_frame_overlay_image
      // return '/userdata/frames/sprencles.png'
    } else {
      return configurationStore.configuration.uisettings.livestream_frameoverlay_image
    }
  } else if (configurationStore.configuration.uisettings.enable_livestream_frameoverlay) {
    return configurationStore.configuration.uisettings.livestream_frameoverlay_image
  } else {
    return ''
  }
})

const showPreview = computed(() => {
  const enabledWhenIdle = configurationStore.configuration.uisettings.enable_livestream_when_idle
  const enabledWhenActive = configurationStore.configuration.uisettings.enable_livestream_when_active
  const machineIdle = !stateStore.state || stateStore.state == 'finished'
  const machineRecord = stateStore.state == 'record'
  const machineCounting = stateStore.state == 'counting'
  const machineCapture = stateStore.state == 'capture'

  // allow user to choose if shown during idle or process only. during record it cannot be disabled because video useful to show while recording
  return (machineIdle && enabledWhenIdle) || ((machineCounting || machineCapture) && enabledWhenActive) || machineRecord
})

const showFrontpage = computed(() => {
  // show if state not defined (no job ongoing or finished)
  return !stateStore.state || stateStore.state == 'finished'
})
const onBtnAdminClick = () => {
  if (adminButtonInvisible.value) {
    btnAdminClickCounter.value++
  } else {
    router.push('/admin')
  }
}
const invokeAction = (action: string, config_index: number) => {
  remoteProcedureCall(`/api/${action}/${config_index}`)
}
const stopRecordingVideo = () => {
  remoteProcedureCall('/api/actions/stop')
}
</script>

<style lang="sass">

// if button shall be invisible, set it to transparent and on mouseover use default cursor, no pointer
.action-button-admin-invisible
  opacity: 0.0
  cursor: default
</style>
