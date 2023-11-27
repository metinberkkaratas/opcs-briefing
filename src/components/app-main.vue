<script lang="ts">
import { useRoute } from 'vue-router'
import { setAllowedBugTracking } from '../bugs'
import { ROOM_PATH } from '../config'
import { historyAddRoom } from '../lib/history'
import { messages } from '../lib/messages'
import { createLinkForRoom, shareLink } from '../lib/share'
import { setup, state } from '../state'
import SeaModal from '../ui/sea-modal.vue'
import AppChat from './app-chat.vue'
import AppSettings from './app-settings.vue'
import AppShare from './app-share.vue'
import AppVideo from './app-video.vue'

export default {
  name: 'AppMain',
  components: {
    AppSettings,
    AppShare,
    AppChat,
    SeaModal,
    AppVideo,
  },
  data() {
    return {
      mode: '',
      settings: false,
      share: false,
      conn: null,

      supportsFullscreen: document.fullscreenEnabled,
      isFullScreen: false,
      fullscreenHandler: null,
      symbol:
        '<svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-linecap="round" stroke-linejoin="round"><path d="M4 12v8a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2v-8"></path><polyline points="16 6 12 2 8 6"></polyline><line x1="12" y1="2" x2="12" y2="15"></line></svg>',
      name: '',
      unreadMessages: false,
      state,
    }
  },
  computed: {
    hasPeers() {
      return Object.keys(state.status).length > 0
    },
    peers() {
      return state.screenshots ? [2, 3] : state.status
    },
    videoAllowed() {
      if (window.webkit != null)
        return this.mode === ''
      return true
    },
  },
  mounted() {
    this.setName()
    this.triggerChatFunctions()

    setTimeout(async () => {
      this.conn = await setup()
    }, 50)
    if (!this.hasPeers && !window.iPhone && state.showInviteOnStart)
      this.mode = 'share'

    this.fullscreenHandler = () => {
      this.isFullScreen = !!document.fullscreenElement
    }
    document.addEventListener('fullscreenchange', this.fullscreenHandler)

    // Remember room name for next visits
    historyAddRoom(state.room)
  },
  beforeUnmount() {
    document.removeEventListener('fullscreenchange', this.fullscreenHandler)
    this.conn?.cleanup()
  },
  methods: {
    doShare() {
      shareLink(createLinkForRoom(state.room))
    },
    doVideo() {
      state.muteVideo = !state.muteVideo
      messages.emit('updateStream')
    },
    doScreen() {
      state.muteScreen = !state.muteScreen
      messages.emit('updateStream')
    },
    doAudio() {
      state.muteAudio = !state.muteAudio
      messages.emit('updateStream')
    },
    doQuit() {
      // eslint-disable-next-line no-alert
      if (confirm('Really quit this session?'))
        location.assign(ROOM_PATH)
    },
    doReload() {
      location.reload()
    },
    doAllow(allow) {
      if (allow)
        setAllowedBugTracking(allow)

      state.requestBugTracking = false
    },
    doToggleFullScreen() {
      if (!document.fullscreenElement) {
        document.documentElement.requestFullscreen()
      }
      else {
        if (document.exitFullscreen)
          document.exitFullscreen()
      }
    },
    doTogglePanel(mode = 'settings') {
      this.mode = (!mode || this.mode === mode) ? '' : mode
    },
    didChangeFullscreen() {},
    toggleChat() {
      if (this.mode === 'chat') {
        this.mode = ''
      }
      else {
        this.unreadMessages = false
        this.mode = 'chat'
        this.focusChatInput()
      }
    },
    updateUserInfo() {
      messages.emit('userInfo', {
        name: this.name,
      })
    },
    triggerChatFunctions() {
      messages.on('newMessage', () => {
        if (this.mode !== 'chat')
          this.unreadMessages = true
      })

      messages.on('userInfoUpdate', ({ peer, data }) => {
        this.peers[
          this.peers.findIndex(el => el.remote === peer.local)
        ].peer.name = data.data.name
      })

      // Update Local Name to Remote peers every 10 seconds for new peers
      messages.on('requestUserInfo', () => {
        this.updateUserInfo()
      })
    },
    setName() {
      const name = localStorage.getItem('name')
//      const route = useRoute()
//      const name = route?.query?.name
      if (name) {
        this.name = name
        messages.emit('userInfo', {
          name,
        })
      }
    },
    focusChatInput() {
      if (
        !/Android|webOS|iPhone|iPad|iPod|Opera Mini/i.test(navigator.userAgent)
      ) {
        setTimeout(() => {
          document.getElementById('message-input')?.focus()
        }, 100)
      }
    },
  },
}
</script>

<template>
  <div class="app hstack">
    <SeaModal
      v-if="mode === 'settings'"
      xclass="panel -left panel-settings"
      active
      :xactive="mode === 'settings'"
      :title="$t('settings.title')"
      @close="mode = ''"
    >
      <AppSettings />
    </SeaModal>

    <div
      class="-fit vstack"
      :data-mode="state.maximized ? 'maximized' : 'default'"
    >
      <div class="-fit stack videos -relative">
        <AppVideo
          v-if="videoAllowed && state.stream"
          id="self"
          :stream="state.stream"
          muted
          :mirrored="state.deviceVideo !== 'desktop'"
          title="Local"
        />

        <AppVideo
          v-for="peer in peers"
          :id="peer.remote"
          :key="peer.remote"
          :stream="peer?.peer?.stream"
          :fingerprint="peer?.peer?.fingerprint"
          :name="peer?.peer?.name"
        />

        <div
          v-if="!state.screenshots && state.requestBugTracking"
          class="message-container -error"
        >
          <div class="message">
            {{ $t("error.ask_to_send_error") }}
            <u @click="doAllow(true)">{{ $t("error.send_allow") }}</u> |
            <u @click="doAllow(false)">{{ $t("error.send_deny") }}</u>
          </div>
        </div>

        <div v-else-if="state.error" class="message-container -error">
          <div class="message">
            {{ state.error }}
            <u @click="doReload">Reload page</u>
          </div>
        </div>

        <div
          v-else-if="
            !hasPeers
              && !state.screenshots
              && mode !== 'share'
              && state.showInviteHint
          "
          class="message-container"
        >
          <div
            class="message"
            v-html="$t('share.message', { symbol })"
          />
        </div>
      </div>

      <div class="tools hstack">
        <button
          v-if="state.showSettings"
          class="tool"
          :class="{ '-active': mode === 'settings' }"
          @click="doTogglePanel('settings')"
        >
          <!--        <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-linecap="round" stroke-linejoin="round"><circle cx="12" cy="12" r="3"></circle><path d="M19.4 15a1.65 1.65 0 0 0 .33 1.82l.06.06a2 2 0 0 1 0 2.83 2 2 0 0 1-2.83 0l-.06-.06a1.65 1.65 0 0 0-1.82-.33 1.65 1.65 0 0 0-1 1.51V21a2 2 0 0 1-2 2 2 2 0 0 1-2-2v-.09A1.65 1.65 0 0 0 9 19.4a1.65 1.65 0 0 0-1.82.33l-.06.06a2 2 0 0 1-2.83 0 2 2 0 0 1 0-2.83l.06-.06a1.65 1.65 0 0 0 .33-1.82 1.65 1.65 0 0 0-1.51-1H3a2 2 0 0 1-2-2 2 2 0 0 1 2-2h.09A1.65 1.65 0 0 0 4.6 9a1.65 1.65 0 0 0-.33-1.82l-.06-.06a2 2 0 0 1 0-2.83 2 2 0 0 1 2.83 0l.06.06a1.65 1.65 0 0 0 1.82.33H9a1.65 1.65 0 0 0 1-1.51V3a2 2 0 0 1 2-2 2 2 0 0 1 2 2v.09a1.65 1.65 0 0 0 1 1.51 1.65 1.65 0 0 0 1.82-.33l.06-.06a2 2 0 0 1 2.83 0 2 2 0 0 1 0 2.83l-.06.06a1.65 1.65 0 0 0-.33 1.82V9a1.65 1.65 0 0 0 1.51 1H21a2 2 0 0 1 2 2 2 2 0 0 1-2 2h-.09a1.65 1.65 0 0 0-1.51 1z"></path></svg> -->
          <svg
            xmlns="http://www.w3.org/2000/svg"
            width="24"
            height="24"
            viewBox="0 0 24 24"
            fill="none"
            stroke="currentColor"
            stroke-linecap="round"
            stroke-linejoin="round"
          >
            <line x1="4" y1="21" x2="4" y2="14" />
            <line x1="4" y1="10" x2="4" y2="3" />
            <line x1="12" y1="21" x2="12" y2="12" />
            <line x1="12" y1="8" x2="12" y2="3" />
            <line x1="20" y1="21" x2="20" y2="16" />
            <line x1="20" y1="12" x2="20" y2="3" />
            <line x1="1" y1="14" x2="7" y2="14" />
            <line x1="9" y1="8" x2="15" y2="8" />
            <line x1="17" y1="16" x2="23" y2="16" />
          </svg>
        </button>
        <div class="-fit">
          <button
            class="tool"
            :class="{ '-off': state.muteVideo }"
            @click="doVideo"
          >
            <svg
              v-if="state.muteVideo"
              xmlns="http://www.w3.org/2000/svg"
              width="24"
              height="24"
              viewBox="0 0 24 24"
              fill="none"
              stroke="currentColor"
              stroke-linecap="round"
              stroke-linejoin="round"
            >
              <path
                d="M16 16v1a2 2 0 0 1-2 2H3a2 2 0 0 1-2-2V7a2 2 0 0 1 2-2h2m5.66 0H14a2 2 0 0 1 2 2v3.34l1 1L23 7v10"
              />
              <line x1="1" y1="1" x2="23" y2="23" />
            </svg>
            <svg
              v-else
              xmlns="http://www.w3.org/2000/svg"
              width="24"
              height="24"
              viewBox="0 0 24 24"
              fill="none"
              stroke="currentColor"
              stroke-linecap="round"
              stroke-linejoin="round"
            >
              <polygon points="23 7 16 12 23 17 23 7" />
              <rect x="1" y="5" width="15" height="14" rx="2" ry="2" />
            </svg>
          </button>
          <button
            class="tool"
            :class="{ '-off': state.muteScreen }"
            @click="doScreen">
            <svg v-if="state.muteScreen"
                 viewBox="0 0 28 28" version="1.1"
                 xmlns="http://www.w3.org/2000/svg"
                 xmlns:xlink="http://www.w3.org/1999/xlink">
              <title>ic_fluent_share_screen_28_regular</title>
              <desc>Created with Sketch.</desc>
              <g id="🔍-System-Icons" stroke="none" stroke-width="1" fill="none" fill-rule="evenodd">
                <g id="ic_fluent_share_screen_28_regular" fill="#212121" fill-rule="nonzero">
                  <path d="M23.75,4.99939 C24.9926,4.99939 26,6.00675 26,7.24939 L26,20.75 C26,21.9926 24.9926,23 23.75,23 L4.25,23 C3.00736,23 2,21.9927 2,20.75 L2,7.24939 C2,6.00675 3.00736,4.99939 4.25,4.99939 L23.75,4.99939 Z M23.75,6.49939 L4.25,6.49939 C3.83579,6.49939 3.5,6.83518 3.5,7.24939 L3.5,20.75 C3.5,21.1642 3.83579,21.5 4.25,21.5 L23.75,21.5 C24.1642,21.5 24.5,21.1642 24.5,20.75 L24.5,7.24939 C24.5,6.83518 24.1642,6.49939 23.75,6.49939 Z M13.9975,8.62102995 C14.1965,8.62102995 14.3874,8.69998 14.5281,8.8407 L17.7826,12.0952 C18.0755,12.3881 18.0755,12.863 17.7826,13.1559 C17.4897,13.4488 17.0148,13.4488 16.7219,13.1559 L14.7477,11.1817 L14.7477,18.6284 C14.7477,19.0426 14.412,19.3784 13.9977,19.3784 C13.5835,19.3784 13.2477,19.0426 13.2477,18.6284 L13.2477,11.1835 L11.2784,13.1555 C10.9858,13.4486 10.5109,13.4489 10.2178,13.1562 C9.92469,12.8636 9.92436,12.3887 10.217,12.0956 L13.467,8.84107 C13.6077,8.70025 13.7985,8.62102995 13.9975,8.62102995 Z" id="🎨-Color">
                  </path>
                </g>
              </g>
            </svg>
            <svg version="1.2" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 28 28" width="28" height="28">
              <title>share-screen-svgrepo-com-svg</title>
              <defs>
                <image  width="28" height="28" id="img1" href="data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABwAAAAcCAMAAABF0y+mAAAAAXNSR0IB2cksfwAAAIRQTFRFAAAAw8PDw8PDw8PDw8PDw8PDw8PDw8PDw8PDw8PDw8PDw8PDw8PDw8PDw8PDw8PDw8PDw8PDw8PDw8PDw8PDw8PDw8PDw8PDw8PDw8PDw8PDw8PDw8PDw8PDw8PDw8PDw8PDw8PDw8PDw8PDw8PDw8PDw8PDw8PDw8PDw8PDw8PDw8PDFIIbFQAAACx0Uk5TACjA/v/CJNPkgn/og0A+U/j2SkT7Nlz82dTd9e7mH73t514jJsG/vry6NzOfQZ6VAAAAi0lEQVR4nGNgGHyAkYmZBQMws7KBJdk5OLkwACc3O1iSmQebeTzMYIqFCy7CywdncrGgSfILCArhkhQWYWERFMUuKSYuIckuJS2EVVJGVo5Jnk1BEaukkjIDkzyDihIOBzGoquF2LX5JoLF4JNXxSGpo4pHU0saQxB7wEEm8UabCii2ymdiwmTfAAACMLwiH/VuSogAAAABJRU5ErkJggg=="/>
              </defs>
              <g id="🔍-System-Icons">
                <g id="ic_fluent_share_screen_28_regular">
                  <path id="🎨-Color" fill-rule="evenodd" class="s0" d="m23.8 5c1.2 0 2.2 1 2.2 2.2v13.6c0 1.2-1 2.2-2.3 2.2h-19.5c-1.2 0-2.2-1-2.2-2.3v-13.5c0-1.2 1-2.2 2.2-2.2zm0 1.5h-19.5c-0.4 0-0.7 0.3-0.7 0.7v13.6c0 0.4 0.3 0.7 0.7 0.7h19.5c0.5 0 0.8-0.3 0.8-0.7v-13.6c0-0.4-0.3-0.7-0.8-0.7zm-9.7 2.1q0.3 0 0.5 0.2l3.3 3.3c0.3 0.3 0.3 0.8 0 1.1-0.3 0.2-0.8 0.2-1.1 0l-2-2v7.4c0 0.4-0.3 0.8-0.7 0.8-0.4 0-0.8-0.4-0.8-0.8v-7.4l-1.9 2c-0.3 0.2-0.8 0.2-1.1 0-0.3-0.3-0.3-0.8 0-1.1l3.3-3.3q0.2-0.2 0.5-0.2z"/>
                  <use id="Katman 1" href="#img1" x="0" y="0"/>
                </g>
              </g>
            </svg>
          </button>
          <button
            class="tool"
            :class="{ '-off': state.muteAudio }"
            @click="doAudio"
          >
            <svg
              v-if="state.muteAudio"
              xmlns="http://www.w3.org/2000/svg"
              width="24"
              height="24"
              viewBox="0 0 24 24"
              fill="none"
              stroke="currentColor"
              stroke-linecap="round"
              stroke-linejoin="round"
            >
              <line x1="1" y1="1" x2="23" y2="23" />
              <path
                d="M9 9v3a3 3 0 0 0 5.12 2.12M15 9.34V4a3 3 0 0 0-5.94-.6"
              />
              <path
                d="M17 16.95A7 7 0 0 1 5 12v-2m14 0v2a7 7 0 0 1-.11 1.23"
              />
              <line x1="12" y1="19" x2="12" y2="23" />
              <line x1="8" y1="23" x2="16" y2="23" />
            </svg>
            <svg
              v-else
              xmlns="http://www.w3.org/2000/svg"
              width="24"
              height="24"
              viewBox="0 0 24 24"
              fill="none"
              stroke="currentColor"
              stroke-linecap="round"
              stroke-linejoin="round"
            >
              <path
                d="M12 1a3 3 0 0 0-3 3v8a3 3 0 0 0 6 0V4a3 3 0 0 0-3-3z"
              />
              <path d="M19 10v2a7 7 0 0 1-14 0v-2" />
              <line x1="12" y1="19" x2="12" y2="23" />
              <line x1="8" y1="23" x2="16" y2="23" />
            </svg>
          </button>
        </div>
        <button
          v-if="state.showFullscreen && supportsFullscreen"
          class="tool"
          @click="doToggleFullScreen"
        >
          <svg
            v-if="!isFullScreen"
            xmlns="http://www.w3.org/2000/svg"
            width="24"
            height="24"
            viewBox="0 0 24 24"
            fill="none"
            stroke="currentColor"
            stroke-width="2"
            stroke-linecap="round"
            stroke-linejoin="round"
          >
            <path
              d="M8 3H5a2 2 0 0 0-2 2v3m18 0V5a2 2 0 0 0-2-2h-3m0 18h3a2 2 0 0 0 2-2v-3M3 16v3a2 2 0 0 0 2 2h3"
            />
          </svg>
          <svg
            v-if="isFullScreen"
            xmlns="http://www.w3.org/2000/svg"
            width="24"
            height="24"
            viewBox="0 0 24 24"
            fill="none"
            stroke="currentColor"
            stroke-width="2"
            stroke-linecap="round"
            stroke-linejoin="round"
          >
            <path
              d="M8 3v3a2 2 0 0 1-2 2H3m18 0h-3a2 2 0 0 1-2-2V3m0 18v-3a2 2 0 0 1 2-2h3M3 16h3a2 2 0 0 1 2 2v3"
            />
          </svg>
        </button>

        <button
          v-if="state.showChat"
          class="tool messageBtn"
          :class="{ '-active': mode === 'chat' }"
          @click="toggleChat()"
        >
          <svg
            xmlns="http://www.w3.org/2000/svg"
            width="24"
            height="24"
            viewBox="0 0 24 24"
            fill="none"
            stroke="currentColor"
            stroke-linecap="round"
            stroke-linejoin="round"
            class="feather feather-message-square"
          >
            <path
              d="M21 15a2 2 0 0 1-2 2H7l-4 4V5a2 2 0 0 1 2-2h14a2 2 0 0 1 2 2z"
            />
          </svg>
          <div v-if="unreadMessages" class="unread-msg" />
        </button>

        <button
          v-if="state.showShare"
          class="tool"
          :class="{ '-active': mode === 'share' }"
          @click="doTogglePanel('share')"
        >
          <svg
            xmlns="http://www.w3.org/2000/svg"
            width="24"
            height="24"
            viewBox="0 0 24 24"
            fill="none"
            stroke="currentColor"
            stroke-linecap="round"
            stroke-linejoin="round"
          >
            <path d="M4 12v8a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2v-8" />
            <polyline points="16 6 12 2 8 6" />
            <line x1="12" y1="2" x2="12" y2="15" />
          </svg>
        </button>
      </div>
    </div>

    <SeaModal
      v-if="mode === 'share'"
      xclass="panel -left panel-share"
      active
      :xactive="mode === 'share'"
      :title="$t('share.title')"
      @close="mode = ''"
    >
      <AppShare />
    </SeaModal>

    <SeaModal
      v-if="mode === 'chat'"
      xclass="panel -left panel-share"
      active
      :xactive="mode === 'chat'"
      title="Chat with others"
      :scrollable="false"
      @close="mode = ''"
    >
      <AppChat :name="name" />
    </SeaModal>
  </div>
</template>
