<template>
  <header
    class="sticky top-0 z-10 bg-white border-b border-solid border-gray-300"
    style="box-shadow: 0 0px 32px -4px rgba(0,0,0,0.15)"
  >
    <div v-if="showPopupNotification" class="bg-brand-yellow-darkest">
      <div class="container mx-auto ">
        <div class="flex flex-wrap px-6 py-4">
          <div class="text-sm w-full">
            Izinkan notifikasi mengirim pesan
            <button class="ml-2 bg-brand-blue text-white font-bold py-1 px-4 rounded focus:outline-none focus:shadow-outline" type="button" @click="allowNotification">
              Allow
            </button>
          </div>
        </div>
      </div>
    </div>
    <div class="container mx-auto p-4">
      <div class="md:hidden">
        <a
          v-if="canGoBack"
          tag="a"
          class="cursor-pointer flex items-center"
          href="javascript:void(0)"
          @click.prevent="onGoBack"
        >
          <FontAwesomeIcon :icon="icon.faChevronLeft" />
          <div class="text-left px-4">
            <p class="text-lg">
              Kembali
            </p>
          </div>
        </a>
        <div v-else class="flex justify-between items-center">
          <div
            class="flex items-center"
          >
            <nuxt-link to="/">
              <img class="block h-10 mx-auto mr-4" src="logo.jpg" alt>
            </nuxt-link>
            <nuxt-link to="/">
              <div class="text-left">
                <p class="text-base">
                  Pusat Informasi &amp; Koordinasi COVID-19
                </p>
                <p class="text-sm text-gray-700">
                  Provinsi Jawa Barat
                </p>
              </div>
            </nuxt-link>
          </div>
        </div>
      </div>
      <div class="hidden md:block md:mx-4">
        <div class="flex justify-between items-center">
          <div
            class="flex items-center"
          >
            <nuxt-link to="/">
              <img class="block h-10 mx-auto mr-4" src="logo.jpg" alt>
            </nuxt-link>
            <nuxt-link to="/">
              <div class="text-left">
                <p class="text-base">
                  Pusat Informasi <br class="inline-block lg:hidden"> &amp;  Koordinasi COVID-19
                </p>
                <p class="text-sm text-gray-700">
                  Provinsi Jawa Barat
                </p>
              </div>
            </nuxt-link>
          </div>
          <ul class="flex flex-row items-center">
            <nuxt-link
              v-for="item in menus"
              :key="item.to"
              tag="li"
              :to="item.to"
              :exact="item.exact"
              class="appbar-menu-item cursor-pointer mx-2 px-2 py-2 hover:bg-gray-200"
            >
              <a>
                {{ item.label }}
              </a>
            </nuxt-link>
            <nuxt-link
              tag="li"
              to="/notifications"
              :exact="true"
              class="cursor-pointer w-12 h-12 flex items-center justify-center rounded-full hover:bg-gray-200 box-content"
            >
              <a class="text-gray-500">
                <FontAwesomeIcon
                  :icon="icon.faBell"
                />
              </a>
            </nuxt-link>
            <nuxt-link
              v-if="enableDownload"
              tag="li"
              :to="downloadAppURL"
              class="cursor-pointer px-4 py-2 text-white text-center ml-2 rounded-md bg-brand-green hover:bg-brand-green-light"
            >
              Download App
            </nuxt-link>
          </ul>
        </div>
      </div>
    </div>
  </header>
</template>

<script>
import { faChevronLeft, faBell } from '@fortawesome/free-solid-svg-icons'
import { db, messaging, FieldValue } from '~/lib/firebase'
export default {
  data () {
    return {
      icon: {
        faChevronLeft,
        faBell
      },
      showPopupNotification: false,
      menus: [
        { to: '/', label: 'Home', exact: true },
        { to: '/articles?tab=jabar', label: 'Berita' },
        { to: '/contact', label: 'Kontak' },
        { to: '/faq', label: 'FAQ' }
      ],
      enableDownload: false,
      downloadAppURL: '#'
    }
  },
  computed: {
    canGoBack () {
      return this.$route.path !== '/'
    },
    pathToGoBack () {
      return this.$store.state.route.from.path
    }
  },
  mounted () {
    this.toggleDownloadAppButton()
    this.checkPermission()
  },
  methods: {
    onGoBack () {
      this.$router.back()
    },
    toggleDownloadAppButton () {
      if (process.client || process.browser) {
        import('../../lib/firebase-client')
          .then((m) => {
            return m ? m.default || m : null
          }).then((module) => {
            if (!module) {
              return
            }
            const { remoteConfig } = module
            remoteConfig.fetchAndActivate()
              .then(() => {
                this.enableDownload = remoteConfig.getValue('download_app_via_web_enabled') === 'true'
                this.downloadAppURL = remoteConfig.getValue('download_app_url')._value
              })
          })
      }
    },
    async checkPermission () {
      if (!messaging) {
        return
      }

      const permission = Notification.permission

      if (permission === 'default') {
        console.log('notification permission default')
        this.showPopupNotification = true
      }

      if (permission === 'granted') {
        const token = await messaging.getToken()
        console.log('notification permission granted')
        this.saveToken(token)
      }
      if (permission === 'denied') {
        console.log('notification permission denied')
      }
    },
    allowNotification () {
      if (!messaging) {
        return
      }
      Notification.requestPermission(async (permission) => {
        console.log({ allowedPermission: permission })
        if (permission === 'granted') {
          const token = await messaging.getToken()
          this.saveToken(token)
        }
        messaging.onTokenRefresh(async () => {
          const token = await messaging.getToken()

          this.saveToken(token)
        })
        this.showPopupNotification = false
      })
    },
    async saveToken (token) {
      const tokenRef = await db.collection('tokens').doc(token)
      const record = await tokenRef.get()

      if (record.exists === false) {
        tokenRef.set({
          token,
          createdAt: FieldValue.serverTimestamp()
        })
      }
    }
  }
}
</script>

<style lang="scss" scoped>
.appbar-menu-item.nuxt-link-active {
  @apply text-green-700;
}
</style>
