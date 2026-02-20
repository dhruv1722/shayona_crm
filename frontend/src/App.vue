<template>
  <FrappeUIProvider>
    <NotPermitted v-if="$route.name === 'Not Permitted'" />
    <Layout class="isolate" v-else-if="session().isLoggedIn">
      <router-view :key="$route.fullPath" />
      <SmartflowDialer />
    </Layout>
    <Dialogs />
    <EventNotificationPopup />
  </FrappeUIProvider>
</template>

<script setup>
import NotPermitted from '@/pages/NotPermitted.vue'
import EventNotificationPopup from '@/components/EventNotificationPopup.vue'
import SmartflowDialer from '@/components/Telephony/SmartflowDialer.vue'
import { Dialogs } from '@/utils/dialogs'
import { sessionStore as session } from '@/stores/session'
import { globalStore } from '@/stores/global'
import { FrappeUIProvider, setConfig } from 'frappe-ui'
import { computed, defineAsyncComponent, onMounted } from 'vue'
import { useSmartflowCallStore } from '@/stores/smartflow'

const callStore = useSmartflowCallStore()

const MobileLayout = defineAsyncComponent(
  () => import('./components/Layouts/MobileLayout.vue'),
)
const DesktopLayout = defineAsyncComponent(
  () => import('./components/Layouts/DesktopLayout.vue'),
)

const Layout = computed(() => {
  if (window.innerWidth < 640) {
    return MobileLayout
  } else {
    return DesktopLayout
  }
})

function handleCallUpdate(data) {
  console.log('[Smartflow] smartflo_call_update received:', data)

  if (!data?.call_id) return

  const savedCallId = sessionStorage.getItem('sf_active_call_id')
  // Is tab ne call nahi kiya — ignore
  if (!savedCallId) return

  // Alag call hai — ignore  
  if (savedCallId !== data.call_id) return

  if (['Completed', 'Failed', 'No Answer', 'Busy', 'Canceled'].includes(data.status)) {
    sessionStorage.removeItem('sf_active_call_id')
  }

  if (callStore.isOpen && callStore.callId === data.call_id) {
    // Popup khula hai, same call — status update
    callStore.updateStatus(data.status)
  } else if (callStore.isOpen && callStore.callId !== data.call_id) {
    return
  } else {
    // Popup band hai — reopen karo
    // User ne X dabaya tha, call abhi bhi chal rahi thi, ab cut hui

    // _last* values check karo — agar same call thi toh name preserve hoga
    const isSameCall = callStore._lastCallId === data.call_id
    const leadName = isSameCall ? callStore._lastLeadName : null
    const to = isSameCall ? callStore._lastTo : data.to
    callStore.open(data.call_id, to, leadName)
    callStore.updateStatus(data.status)
  }
}

function setupSocket(socket) {
  socket.off('smartflo_call_update', handleCallUpdate)
  socket.on('smartflo_call_update', handleCallUpdate)
  console.log('[Smartflow] Listener registered ✅')
}

onMounted(() => {
  const { $socket } = globalStore()

  if (!$socket) {
    console.warn('[Smartflow] $socket not found')
    return
  }

  if ($socket.connected) {
    setupSocket($socket)
  } else {
    $socket.on('connect', () => {
      console.log('[Smartflow] Socket connected!')
      setupSocket($socket)
    })
  }
})

setConfig('systemTimezone', window.timezone?.system || null)
setConfig('localTimezone', window.timezone?.user || null)
</script>