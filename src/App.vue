<template>
  <div id="app" :class="{ 'user-select-none': userSelectNone }">
    <Scrollbar v-show="!showLyrics" ref="scrollbar" />
    <Navbar v-show="showNavbar" ref="navbar" />
    <main
      ref="main"
      :style="{ overflow: enableScrolling ? 'auto' : 'hidden' }"
      @scroll="handleScroll"
    >
      <keep-alive>
        <router-view v-if="$route.meta.keepAlive"></router-view>
      </keep-alive>
      <router-view v-if="!$route.meta.keepAlive"></router-view>
    </main>
    <transition name="slide-up">
      <Player v-if="enablePlayer" v-show="showPlayer" ref="player" />
    </transition>
    <Toast />
    <ModalAddTrackToPlaylist />
    <ModalNewPlaylist />
    <MusicPlay v-if="enablePlayer" ref="mainMusicPlayRef" />
  </div>
</template>

<script>
import ModalAddTrackToPlaylist from './components/ModalAddTrackToPlaylist.vue';
import ModalNewPlaylist from './components/ModalNewPlaylist.vue';
import Scrollbar from './components/Scrollbar.vue';
import Navbar from './components/Navbar.vue';
import Player from './components/Player.vue';
import Toast from './components/Toast.vue';
import { ipcRenderer } from './electron/ipcRenderer';
import { isAccountLoggedIn, isLooseLoggedIn } from '@/utils/auth';
import MusicPlay from './views/musicPlay.vue';
import { mapState, mapMutations } from 'vuex';
import { isMac, isLinux } from './utils/platform';

export default {
  name: 'App',
  components: {
    Navbar,
    Player,
    Toast,
    ModalAddTrackToPlaylist,
    ModalNewPlaylist,
    MusicPlay,
    Scrollbar,
  },
  data() {
    return {
      isElectron: process.env.IS_ELECTRON, // true || undefined
      userSelectNone: false,
    };
  },
  computed: {
    ...mapState([
      'showLyrics',
      'player',
      'enableScrolling',
      'settings',
      'localMusic',
      'enabledVirtualScroll',
    ]),
    localMusicPath() {
      return this.settings.localMusicFolderPath;
    },
    isAccountLoggedIn() {
      return isAccountLoggedIn();
    },
    showPlayer() {
      return (
        [
          'mv',
          'loginUsername',
          'login',
          'loginAccount',
          'lastfmCallback',
        ].includes(this.$route.name) === false
      );
    },
    enablePlayer() {
      return this.player.enabled && this.$route.name !== 'lastfmCallback';
    },
    showNavbar() {
      return this.$route.name !== 'lastfmCallback';
    },
    virtualScroll() {
      return this.enabledVirtualScroll;
    },
  },
  watch: {
    localMusicPath(value) {
      if (!value) return;
      this.deleteLocalMusic();
      const render = require('electron').ipcRenderer;
      render.send('currentLocalMusic', []);
      render.send('msgScanLocalMusic', value);
    },
    virtualScroll(value) {
      if (value) {
        this.$store.commit('enableScrolling', false);
      } else {
        this.$store.commit('enableScrolling', true);
      }
    },
  },
  created() {
    if (this.isElectron) {
      ipcRenderer(this);
      const render = require('electron').ipcRenderer;
      const show_menu = isMac
        ? this.settings.showLyricsMenu &&
          !this.settings.showStatusBarLyric &&
          !this.settings.showControl
        : true;
      if (show_menu) {
        render.send('switchRepeatMode', this.player.repeatMode);
        render.send('switchShuffle', this.player.shuffle);
      }
      if (isLinux) {
        render.invoke('checkExtensionStatus').then(res => {
          this.$store.commit('updateDBusStatus', res);
        });
      }
    }
    if (isMac && this.isElectron) {
      const { initMacStatusbarLyric } = require('./utils/macStatusBarLyric');
      initMacStatusbarLyric();
    }
    window.addEventListener('keydown', this.handleKeydown);
    this.fetchData();
  },
  methods: {
    ...mapMutations(['deleteLocalMusic']),
    fetchData() {
      if (!isLooseLoggedIn()) return;
      this.$store.dispatch('fetchLikedSongs');
      this.$store.dispatch('fetchLikedSongsWithDetails');
      this.$store.dispatch('fetchLikedPlaylist');
      if (isAccountLoggedIn()) {
        this.$store.dispatch('fetchLikedAlbums');
        this.$store.dispatch('fetchLikedArtists');
        this.$store.dispatch('fetchLikedMVs');
        this.$store.dispatch('fetchCloudDisk');
      }
    },
    handleScroll() {
      this.$refs.scrollbar.handleScroll();
    },
  },
};
</script>

<style lang="scss">
#app {
  width: 100%;
  transition: all 0.4s;
}

main {
  position: fixed;
  top: 0;
  bottom: 0;
  right: 0;
  left: 0;
  overflow: auto;
  padding: 64px 10vw 96px 10vw;
  box-sizing: border-box;
  scrollbar-width: none; // firefox
}

@media (max-width: 1336px) {
  main {
    padding: 64px 5vw 96px 5vw;
  }
}

main::-webkit-scrollbar {
  width: 0px;
}

.slide-up-enter-active,
.slide-up-leave-active {
  transition: transform 0.4s;
}
.slide-up-enter,
.slide-up-leave-to {
  transform: translateY(100%);
}
.contextMenu {
  width: 0;
}
</style>
