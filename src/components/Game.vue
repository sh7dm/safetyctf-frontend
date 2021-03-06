<template>
<v-app :dark="dark" :light="!dark">
  <v-toolbar>
    <v-toolbar-title>⚡️SafetyCTF👨‍💻👩‍💻</v-toolbar-title>
    <v-spacer></v-spacer>
    <v-toolbar-items class="hidden-sm-and-down">
      <v-btn flat to="/"><v-icon>home</v-icon>{{ $t('message.home') }}</v-btn>
      <v-btn flat @click="switchDark"><v-icon>invert_colors</v-icon>{{ dark ? $t('message.dark') : $t('message.light') }}</v-btn>
      <lang-switch></lang-switch>
      <v-btn flat @click="(isSubscribed ? unsubscribe : subscribe)()" v-if="auth.$auth"><v-icon :class="isSubscribed ? '' : 'animated infinite tada'">notifications_none</v-icon>{{ isSubscribed ? 'Unsubscribe' : 'Subscribe' }}</v-btn>
      <v-btn flat @click="logout" v-if="auth.$auth"><v-icon>exit_to_app</v-icon>{{ $t('message.logout') }}</v-btn>
    </v-toolbar-items>
  </v-toolbar>
  <v-navigation-drawer fixed app>
    <v-toolbar flat>
      <v-list>
        <v-list-tile>
          <!--<v-list-tile-title class="title">
            SafetyCTF
          </v-list-tile-title>-->

          <logo width='225' heigth='62.5' :isVertical='false'></logo>
        </v-list-tile>
      </v-list>
    </v-toolbar>
    <v-divider></v-divider>
    <v-list dense class="pt-0">
      <v-list-tile to="/game/myaccount">
        <v-list-tile-action>
          <v-icon>fingerprint</v-icon>
        </v-list-tile-action>
        <v-list-tile-content>
          <v-list-tile-title>{{ $t('message.myaccount') }}</v-list-tile-title>
        </v-list-tile-content>
      </v-list-tile>
      <v-list-tile to="/game/tasks">
        <v-list-tile-action>
          <v-icon>code</v-icon>
        </v-list-tile-action>
        <v-list-tile-content>
          <v-list-tile-title>{{ $t('message.tasks') }}</v-list-tile-title>
        </v-list-tile-content>
      </v-list-tile>
      <v-list-tile v-if='auth.$auth && auth.$isAdmin' to="/game/admin">
        <v-list-tile-action>
          <v-icon>supervisor_account</v-icon>
        </v-list-tile-action>
        <v-list-tile-content>
          <v-list-tile-title>{{ $t('message.admin') }}</v-list-tile-title>
        </v-list-tile-content>
      </v-list-tile>
      <v-list-tile to="/game/leaderboard">
        <v-list-tile-action>
          <v-icon>stars</v-icon>
        </v-list-tile-action>
        <v-list-tile-content>
          <v-list-tile-title>{{ $t('message.leaderboard') }}</v-list-tile-title>
        </v-list-tile-content>
      </v-list-tile>
    </v-list>
  </v-navigation-drawer>
  <v-content>
    <transition enter-active-class='animated bounceInLeft' leave-active-class='animated bounceOutRight' mode='out-in' duration='300'>
      <router-view></router-view>
    </transition>
  </v-content>
</v-app>
</template>

<script>
import LangSwitch from './Game/LangSwitch'
import Logo from './Logo'
import auth from '../auth'
import firebase from 'firebase'
import swal from 'sweetalert2'
import lscache from 'lscache'

// Initialize Firebase
firebase.initializeApp({ messagingSenderId: '652086280144' })
const messaging = firebase.messaging()
messaging.usePublicVapidKey('BL0N5Q6Ud09Gm8JLIuX_DEi1Iq2_ovXPX2WB_iZWSEZH1kK7GVa3h1S8-AfXfTpY3bQGrsrnhHHaukU98OUkzrY')

export default {
  components: { LangSwitch, Logo },
  data () {
    return {
      dark: false,
      isSubscribed: false,
      auth
    }
  },
  i18n: {
    messages: {
      en: {
        message: {
          myaccount: 'My account',
          tasks: 'Tasks',
          admin: 'Admin panel',
          leaderboard: 'Leaderboard',
          logout: 'Log out',
          dark: 'Dark theme',
          light: 'Light theme',
          home: 'Home'
        }
      },
      ru: {
        message: {
          myaccount: 'Мой аккаунт',
          tasks: 'Задания',
          admin: 'Панель управления',
          leaderboard: 'Таблица лидеров',
          logout: 'Выйти',
          dark: 'Тёмная тема',
          light: 'Светлая тема',
          home: 'Главная страница'
        }
      }
    }
  },
  mounted () {
    this.dark = localStorage.getItem('dark') === 'true'

    const { refreshToken } = this
    this.$el.onclick = function () {
      refreshToken()
    }

    if (localStorage.getItem('pushTokenSentToServer')) this.isSubscribed = true
    else this.isSubscribed = false

    if ('Notification' in window && Notification.permission === 'granted' && this.isSubscribed && this.auth.$auth) {
      this.subscribe()
    }

    messaging.onMessage(function (payload) {
      console.log('Message received. ', payload)
      swal(payload.notification.title, payload.notification.body, 'success')
    })
  },
  methods: {
    unsubscribe () {
      this.sendTokenToServer('')
      this.setTokenSentToServer(false)
    },
    subscribe () {
      const { setTokenSentToServer, sendTokenToServer } = this

      messaging.requestPermission().then(function () {
        console.log('Notification permission granted.')
        messaging.getToken().then(async function (currentToken) {
          if (currentToken) {
            await sendTokenToServer(currentToken)
          } else {
            // Show permission request.
            console.log('No Instance ID token available. Request permission to generate one.')
            // Show permission UI.
            // updateUIForPushPermissionRequired()
            setTokenSentToServer(false)
          }
        }).catch(function (err) {
          console.log('An error occurred while retrieving token. ', err)
          // showToken('Error retrieving Instance ID token. ', err)
          setTokenSentToServer(false)
        })

        messaging.onTokenRefresh(function () {
          messaging.getToken().then(async function (refreshedToken) {
            console.log('Token refreshed.')
            // Indicate that the new Instance ID token has not yet been sent to the
            // app server.
            setTokenSentToServer(false)
            // Send Instance ID token to app server.
            await sendTokenToServer(refreshedToken)
            // ...
          }).catch(function (err) {
            console.log('Unable to retrieve refreshed token ', err)
            // showToken('Unable to retrieve refreshed token ', err)
          })
        })
      }).catch(function (err) {
        console.log('Unable to get permission to notify.', err)
      })
    },
    sendTokenToServer (currentToken) {
      this.setTokenSentToServer(currentToken)

      return this.$http.post('/push/sub', {
        jwt: this.$getToken(),
        token: currentToken
      })
    },
    setTokenSentToServer (currentToken) {
      if (currentToken) this.isSubscribed = true
      else this.isSubscribed = false

      localStorage.setItem('pushTokenSentToServer', currentToken)
    },

    switchDark () {
      this.dark = !this.dark
      localStorage.setItem('dark', this.dark) // Save to localStorage
    },
    async refreshToken () {
      if (this.$getTokenExp() && this.$getToken()) {
        try {
          const { data } = await this.$http.post('/auth/refreshToken', { token: this.$getToken() })

          if (data.code === 200) {
            lscache.set('token', data.token, 60)
            lscache.set('token-exp', 1, 55)
          }
        } catch (e) {
          console.log(e)
        }
      }
    },
    logout () {
      lscache.remove('token')
      lscache.remove('token-exp')
    }
  }
}
</script>
