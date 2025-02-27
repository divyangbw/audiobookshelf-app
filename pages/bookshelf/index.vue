<template>
  <div class="w-full h-full min-h-full relative">
    <div v-if="!loading" class="w-full" :class="{ 'py-6': altViewEnabled }">
      <template v-for="(shelf, index) in shelves">
        <bookshelf-shelf :key="shelf.id" :label="shelf.label" :entities="shelf.entities" :type="shelf.type" :style="{ zIndex: shelves.length - index }" />
      </template>
    </div>

    <div v-if="!shelves.length && !loading" class="absolute top-0 left-0 w-full h-full flex items-center justify-center">
      <div>
        <p class="mb-4 text-center text-xl">
          Bookshelf empty
          <span v-show="user">
            for library
            <strong>{{ currentLibraryName }}</strong>
          </span>
        </p>
        <div class="w-full" v-if="!user">
          <div class="flex justify-center items-center mb-3">
            <span class="material-icons text-error text-lg">cloud_off</span>
            <p class="pl-2 text-error text-sm">Audiobookshelf server not connected.</p>
          </div>
        </div>
        <div class="flex justify-center">
          <ui-btn v-if="!user" small @click="$router.push('/connect')" class="w-32">Connect</ui-btn>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  props: {
    loading: Boolean
  },
  data() {
    return {
      shelves: [],
      isSettingsLoaded: false,
      isFirstNetworkConnection: true,
      lastServerFetch: 0,
      lastServerFetchLibraryId: null,
      lastLocalFetch: 0,
      localLibraryItems: []
    }
  },
  watch: {
    networkConnected(newVal) {
      // Update shelves when network connect status changes
      console.log(`[categories] Network changed to ${newVal} - fetch categories. ${this.lastServerFetch}/${this.lastLocalFetch}`)

      if (newVal) {
        // Fetch right away the first time network connects
        if (this.isFirstNetworkConnection) {
          this.isFirstNetworkConnection = false
          console.log(`[categories] networkConnected true first network connection. lastServerFetch=${this.lastServerFetch}`)
          this.fetchCategories()
          return
        }

        setTimeout(() => {
          // Using timeout because making this fetch as soon as network gets connected will often fail on Android
          console.log(`[categories] networkConnected true so fetching categories. lastServerFetch=${this.lastServerFetch}`)
          this.fetchCategories()
        }, 4000)
      } else {
        console.log(`[categories] networkConnected false so fetching categories`)
        this.fetchCategories()
      }
    }
  },
  computed: {
    user() {
      return this.$store.state.user.user
    },
    networkConnected() {
      return this.$store.state.networkConnected
    },
    isIos() {
      return this.$platform === 'ios'
    },
    currentLibraryName() {
      return this.$store.getters['libraries/getCurrentLibraryName']
    },
    currentLibraryId() {
      return this.$store.state.libraries.currentLibraryId
    },
    currentLibraryMediaType() {
      return this.$store.getters['libraries/getCurrentLibraryMediaType']
    },
    currentLibraryIsPodcast() {
      return this.currentLibraryMediaType === 'podcast'
    },
    altViewEnabled() {
      return this.$store.getters['getAltViewEnabled']
    },
    localMediaProgress() {
      return this.$store.state.globals.localMediaProgress
    },
    isLoading: {
      get() {
        return this.loading
      },
      set(val) {
        this.$emit('update:loading', val)
      }
    }
  },
  methods: {
    async getLocalMediaItemCategories() {
      const localMedia = await this.$db.getLocalLibraryItems()
      if (!localMedia || !localMedia.length) return []

      const categories = []
      const books = []
      const podcasts = []
      const booksContinueListening = []
      const podcastEpisodesContinueListening = []
      localMedia.forEach((item) => {
        if (item.mediaType == 'book') {
          item.progress = this.$store.getters['globals/getLocalMediaProgressById'](item.id)
          if (item.progress && !item.progress.isFinished && item.progress.progress > 0) booksContinueListening.push(item)
          books.push(item)
        } else if (item.mediaType == 'podcast') {
          const podcastEpisodeItemCloner = { ...item }
          item.media.episodes = item.media.episodes.map((ep) => {
            ep.progress = this.$store.getters['globals/getLocalMediaProgressById'](item.id, ep.id)
            if (ep.progress && !ep.progress.isFinished && ep.progress.progress > 0) {
              podcastEpisodesContinueListening.push({
                ...podcastEpisodeItemCloner,
                recentEpisode: ep
              })
            }
            return ep
          })
          podcasts.push(item)
        }
      })

      // Local continue listening shelves, only shown offline
      if (booksContinueListening.length) {
        categories.push({
          id: 'local-books-continue',
          label: 'Continue Books',
          type: 'book',
          localOnly: true,
          entities: booksContinueListening.sort((a, b) => {
            if (a.progress && b.progress) {
              return b.progress.lastUpdate > a.progress.lastUpdate ? 1 : -1
            }
            return 0
          })
        })
      }
      if (podcastEpisodesContinueListening.length) {
        categories.push({
          id: 'local-episodes-continue',
          label: 'Continue Episodes',
          type: 'episode',
          localOnly: true,
          entities: podcastEpisodesContinueListening.sort((a, b) => {
            if (a.recentEpisode.progress && b.recentEpisode.progress) {
              return b.recentEpisode.progress.lastUpdate > a.recentEpisode.progress.lastUpdate ? 1 : -1
            }
            return 0
          })
        })
      }

      // Local books and local podcast shelves
      if (books.length) {
        categories.push({
          id: 'local-books',
          label: 'Local Books',
          type: 'book',
          entities: books.sort((a, b) => {
            if (a.progress && a.progress.isFinished) return 1
            else if (b.progress && b.progress.isFinished) return -1
            else if (a.progress && b.progress) {
              return b.progress.lastUpdate > a.progress.lastUpdate ? 1 : -1
            }
            return 0
          })
        })
      }
      if (podcasts.length) {
        categories.push({
          id: 'local-podcasts',
          label: 'Local Podcasts',
          type: 'podcast',
          entities: podcasts
        })
      }

      return categories
    },
    async fetchCategories() {
      console.log(`[categories] fetchCategories networkConnected=${this.networkConnected}, lastServerFetch=${this.lastServerFetch}, lastLocalFetch=${this.lastLocalFetch}`)

      // TODO: Find a better way to keep the shelf up-to-date with local vs server library because this is a disaster
      if (this.user && this.currentLibraryId && this.networkConnected) {
        if (this.lastServerFetch && Date.now() - this.lastServerFetch < 5000 && this.lastServerFetchLibraryId == this.currentLibraryId) {
          console.log(`[categories] fetchCategories server fetch was ${Date.now() - this.lastServerFetch}ms ago so not doing it.`)
          return
        } else {
          console.log(`[categories] fetchCategories fetching from server. Last was ${this.lastServerFetch ? Date.now() - this.lastServerFetch + 'ms' : 'Never'} ago. lastServerFetchLibraryId=${this.lastServerFetchLibraryId} and currentLibraryId=${this.currentLibraryId}`)
          this.lastServerFetchLibraryId = this.currentLibraryId
          this.lastServerFetch = Date.now()
          this.lastLocalFetch = 0
        }
      } else {
        if (this.lastLocalFetch && Date.now() - this.lastLocalFetch < 5000) {
          console.log(`[categories] fetchCategories local fetch was ${Date.now() - this.lastLocalFetch}ms ago so not doing it.`)
          return
        } else {
          console.log(`[categories] fetchCategories fetching from local. Last was ${this.lastLocalFetch ? Date.now() - this.lastLocalFetch + 'ms' : 'Never'} ago`)
          this.lastServerFetchLibraryId = null
          this.lastServerFetch = 0
          this.lastLocalFetch = Date.now()
        }
      }

      this.isLoading = true
      this.shelves = []

      if (this.user && this.currentLibraryId && this.networkConnected) {
        this.localLibraryItems = await this.$db.getLocalLibraryItems()
        const localCategories = await this.getLocalMediaItemCategories()
        const categories = await this.$axios.$get(`/api/libraries/${this.currentLibraryId}/personalized?minified=1`).catch((error) => {
          console.error('[categories] Failed to fetch categories', error)
          return []
        })
        if (!categories.length) {
          // Failed to load categories so use local shelves
          console.warn(`[categories] Failed to get server categories so using local categories`)
          this.shelves = localCategories
          this.lastServerFetch = 0
          this.lastLocalFetch = Date.now()
          this.isLoading = false
          console.log('[categories] Local shelves set from failure', this.shelves.length, this.lastLocalFetch)
          return
        }

        this.shelves = categories.map((cat) => {
          if (cat.type == 'book' || cat.type == 'podcast' || cat.type == 'episode') {
            // Map localLibraryItem to entities
            cat.entities = cat.entities.map((entity) => {
              const localLibraryItem = this.localLibraryItems.find((lli) => {
                return lli.libraryItemId == entity.id
              })
              if (localLibraryItem) {
                entity.localLibraryItem = localLibraryItem
              }
              return entity
            })
          }
          return cat
        })

        // TODO: iOS has its own implementation of this. Android & iOS should be consistent here.
        if (!this.isIos) {
          this.openMediaPlayerWithMostRecentListening()
        }

        // Only add the local shelf with the same media type
        const localShelves = localCategories.filter((cat) => cat.type === this.currentLibraryMediaType && !cat.localOnly)
        this.shelves.push(...localShelves)
        console.log('[categories] Server shelves set', this.shelves.length, this.lastServerFetch)
      } else {
        // Offline only local
        this.localLibraryItems = await this.$db.getLocalLibraryItems()
        const localCategories = await this.getLocalMediaItemCategories()
        this.shelves = localCategories
        console.log('[categories] Local shelves set', this.shelves.length, this.lastLocalFetch)
      }

      this.isLoading = false
    },
    async waitForSettings() {
      // Wait up to 3 seconds
      for (let i = 0; i < 6; i++) {
        if (this.isSettingsLoaded) return true
        await new Promise((resolve) => setTimeout(resolve, 500))
      }
      return false
    },
    async openMediaPlayerWithMostRecentListening() {
      // If we don't already have a player open
      // Try opening the first book from continue-listening without playing it
      if (this.$store.state.playerLibraryItemId || !this.$store.state.isFirstAudioLoad) return
      this.$store.commit('setIsFirstAudioLoad', false) // Only run this once on app launch

      // Wait for settings to load to prevent race condition when setting playback speed.
      if (!this.isSettingsLoaded) {
        await this.waitForSettings()
      }

      const continueListeningShelf = this.shelves.find((cat) => cat.id === 'continue-listening')
      const mostRecentEntity = continueListeningShelf?.entities?.find((li) => li.media?.audioTracks?.length || li.media?.numTracks)
      if (mostRecentEntity) {
        const playObject = {
          libraryItemId: mostRecentEntity.id,
          episodeId: mostRecentEntity.recentEpisode?.id || null,
          paused: true
        }

        // Check if there is a local copy
        if (mostRecentEntity.localLibraryItem) {
          if (mostRecentEntity.recentEpisode) {
            // Check if the podcast episode has a local copy
            const localEpisode = mostRecentEntity.localLibraryItem.media.episodes.find((ep) => ep.serverEpisodeId === mostRecentEntity.recentEpisode.id)
            if (localEpisode) {
              playObject.libraryItemId = mostRecentEntity.localLibraryItem.id
              playObject.episodeId = localEpisode.id
              playObject.serverLibraryItemId = mostRecentEntity.id
              playObject.serverEpisodeId = mostRecentEntity.recentEpisode.id
            }
          } else {
            playObject.libraryItemId = mostRecentEntity.localLibraryItem.id
            playObject.serverLibraryItemId = mostRecentEntity.id
          }
        }
        this.$eventBus.$emit('play-item', playObject)
      }
    },
    libraryChanged() {
      if (this.currentLibraryId) {
        console.log(`[categories] libraryChanged so fetching categories`)
        this.fetchCategories()
      }
    },
    audiobookAdded(audiobook) {
      // TODO: Check if audiobook would be on this shelf
      if (!this.search) {
        this.fetchCategories()
      }
    },
    audiobookUpdated(audiobook) {
      this.shelves.forEach((shelf) => {
        if (shelf.type === 'books') {
          shelf.entities = shelf.entities.map((ent) => {
            if (ent.id === audiobook.id) {
              return audiobook
            }
            return ent
          })
        } else if (shelf.type === 'series') {
          shelf.entities.forEach((ent) => {
            ent.books = ent.books.map((book) => {
              if (book.id === audiobook.id) return audiobook
              return book
            })
          })
        }
      })
    },
    removeBookFromShelf(audiobook) {
      this.shelves.forEach((shelf) => {
        if (shelf.type === 'books') {
          shelf.entities = shelf.entities.filter((ent) => {
            return ent.id !== audiobook.id
          })
        } else if (shelf.type === 'series') {
          shelf.entities.forEach((ent) => {
            ent.books = ent.books.filter((book) => {
              return book.id !== audiobook.id
            })
          })
        }
      })
    },
    settingsUpdated() {
      this.isSettingsLoaded = true
    },
    initListeners() {
      this.$eventBus.$on('library-changed', this.libraryChanged)
      this.$eventBus.$on('user-settings', this.settingsUpdated)
    },
    removeListeners() {
      this.$eventBus.$off('library-changed', this.libraryChanged)
      this.$eventBus.$off('user-settings', this.settingsUpdated)
    }
  },
  mounted() {
    this.initListeners()
    console.log(`[categories] mounted so fetching categories`)
    this.fetchCategories()
  },
  beforeDestroy() {
    this.removeListeners()
  }
}
</script>