<template>
  <div id="app">
    <navbar-top />
    <vue-progress-bar class="fd-progress-bar" />
    <transition name="fade">
      <!-- Setting v-show to true on the router-view tag avoids jumpiness during transitions -->
      <router-view v-show="true" />
    </transition>
    <modal-dialog-remote-pairing :show="pairing_active" @close="pairing_active = false" />
    <notifications v-show="!show_burger_menu" />
    <navbar-bottom />
    <div class="fd-overlay-fullscreen" v-show="show_burger_menu || show_player_menu"
        @click="show_burger_menu = show_player_menu = false"></div>
  </div>
</template>

<script>
import NavbarTop from '@/components/NavbarTop'
import NavbarBottom from '@/components/NavbarBottom'
import Notifications from '@/components/Notifications'
import ModalDialogRemotePairing from '@/components/ModalDialogRemotePairing'
import webapi from '@/webapi'
import * as types from '@/store/mutation_types'
import ReconnectingWebSocket from 'reconnectingwebsocket'
import moment from 'moment'

export default {
  name: 'App',
  components: { NavbarTop, NavbarBottom, Notifications, ModalDialogRemotePairing },
  template: '<App/>',

  data () {
    return {
      token_timer_id: 0,
      reconnect_attempts: 0,
      pairing_active: false
    }
  },

  computed: {
    show_burger_menu: {
      get () {
        return this.$store.state.show_burger_menu
      },
      set (value) {
        this.$store.commit(types.SHOW_BURGER_MENU, value)
      }
    },
    show_player_menu: {
      get () {
        return this.$store.state.show_player_menu
      },
      set (value) {
        this.$store.commit(types.SHOW_PLAYER_MENU, value)
      }
    }
  },

  created: function () {
    moment.locale(navigator.language)
    this.connect()

    //  Start the progress bar on app start
    this.$Progress.start()

    //  Hook the progress bar to start before we move router-view
    this.$router.beforeEach((to, from, next) => {
      if (to.meta.show_progress) {
        if (to.meta.progress !== undefined) {
          const meta = to.meta.progress
          this.$Progress.parseMeta(meta)
        }
        this.$Progress.start()
      }
      next()
    })

    //  hook the progress bar to finish after we've finished moving router-view
    this.$router.afterEach((to, from) => {
      if (to.meta.show_progress) {
        this.$Progress.finish()
      }
    })
  },

  methods: {
    connect: function () {
      this.$store.dispatch('add_notification', { text: 'Connecting to forked-daapd', type: 'info', topic: 'connection', timeout: 2000 })

      webapi.config().then(({ data }) => {
        this.$store.commit(types.UPDATE_CONFIG, data)
        this.$store.commit(types.HIDE_SINGLES, data.hide_singles)
        document.title = data.library_name

        this.open_ws()
        this.$Progress.finish()
      }).catch(() => {
        this.$store.dispatch('add_notification', { text: 'Failed to connect to forked-daapd', type: 'danger', topic: 'connection' })
      })
    },

    open_ws: function () {
      if (this.$store.state.config.websocket_port <= 0) {
        this.$store.dispatch('add_notification', { text: 'Missing websocket port', type: 'danger' })
        return
      }

      const vm = this

      var protocol = 'ws://'
      if (window.location.protocol === 'https:') {
        protocol = 'wss://'
      }

      var wsUrl = protocol + window.location.hostname + ':' + vm.$store.state.config.websocket_port
      if (process.env.NODE_ENV === 'development' && process.env.VUE_APP_WEBSOCKET_SERVER) {
        // If we are running in the development server, use the websocket url configured in .env.development
        wsUrl = process.env.VUE_APP_WEBSOCKET_SERVER
      }

      var socket = new ReconnectingWebSocket(
        wsUrl,
        'notify',
        { reconnectInterval: 3000 }
      )

      socket.onopen = function () {
        vm.$store.dispatch('add_notification', { text: 'Connection to server established', type: 'primary', topic: 'connection', timeout: 2000 })
        vm.reconnect_attempts = 0
        socket.send(JSON.stringify({ notify: ['update', 'database', 'player', 'options', 'outputs', 'volume', 'spotify', 'lastfm', 'pairing'] }))

        vm.update_outputs()
        vm.update_player_status()
        vm.update_library_stats()
        vm.update_settings()
        vm.update_queue()
        vm.update_spotify()
        vm.update_lastfm()
        vm.update_pairing()
      }
      socket.onclose = function () {
        // vm.$store.dispatch('add_notification', { text: 'Connection closed', type: 'danger', timeout: 2000 })
      }
      socket.onerror = function () {
        vm.reconnect_attempts++
        vm.$store.dispatch('add_notification', { text: 'Connection lost. Reconnecting ... (' + vm.reconnect_attempts + ')', type: 'danger', topic: 'connection' })
      }
      socket.onmessage = function (response) {
        var data = JSON.parse(response.data)
        if (data.notify.includes('update') || data.notify.includes('database')) {
          vm.update_library_stats()
        }
        if (data.notify.includes('player') || data.notify.includes('options') || data.notify.includes('volume')) {
          vm.update_player_status()
        }
        if (data.notify.includes('outputs') || data.notify.includes('volume')) {
          vm.update_outputs()
        }
        if (data.notify.includes('queue')) {
          vm.update_queue()
        }
        if (data.notify.includes('spotify')) {
          vm.update_spotify()
        }
        if (data.notify.includes('lastfm')) {
          vm.update_lastfm()
        }
        if (data.notify.includes('pairing')) {
          vm.update_pairing()
        }
      }
    },

    update_library_stats: function () {
      webapi.library_stats().then(({ data }) => {
        this.$store.commit(types.UPDATE_LIBRARY_STATS, data)
      })
      webapi.library_count('media_kind is audiobook').then(({ data }) => {
        this.$store.commit(types.UPDATE_LIBRARY_AUDIOBOOKS_COUNT, data)
      })
      webapi.library_count('media_kind is podcast').then(({ data }) => {
        this.$store.commit(types.UPDATE_LIBRARY_PODCASTS_COUNT, data)
      })
    },

    update_outputs: function () {
      webapi.outputs().then(({ data }) => {
        this.$store.commit(types.UPDATE_OUTPUTS, data.outputs)
      })
    },

    update_player_status: function () {
      webapi.player_status().then(({ data }) => {
        this.$store.commit(types.UPDATE_PLAYER_STATUS, data)
      })
    },

    update_queue: function () {
      webapi.queue().then(({ data }) => {
        this.$store.commit(types.UPDATE_QUEUE, data)
      })
    },

    update_settings: function () {
      webapi.settings().then(({ data }) => {
        this.$store.commit(types.UPDATE_SETTINGS, data)
      })
    },

    update_lastfm: function () {
      webapi.lastfm().then(({ data }) => {
        this.$store.commit(types.UPDATE_LASTFM, data)
      })
    },

    update_spotify: function () {
      webapi.spotify().then(({ data }) => {
        this.$store.commit(types.UPDATE_SPOTIFY, data)

        if (this.token_timer_id > 0) {
          window.clearTimeout(this.token_timer_id)
          this.token_timer_id = 0
        }
        if (data.webapi_token_expires_in > 0 && data.webapi_token) {
          this.token_timer_id = window.setTimeout(this.update_spotify, 1000 * data.webapi_token_expires_in)
        }
      })
    },

    update_pairing: function () {
      webapi.pairing().then(({ data }) => {
        this.$store.commit(types.UPDATE_PAIRING, data)
        this.pairing_active = data.active
      })
    },

    update_is_clipped: function () {
      if (this.show_burger_menu || this.show_player_menu) {
        document.querySelector('html').classList.add('is-clipped')
      } else {
        document.querySelector('html').classList.remove('is-clipped')
      }
    }
  },

  watch: {
    'show_burger_menu' () {
      this.update_is_clipped()
    },
    'show_player_menu' () {
      this.update_is_clipped()
    }
  }
}
</script>

<style>
</style>
