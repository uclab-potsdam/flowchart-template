<template>
  <!-- todo: reactivate when adding narration.mp3
  <audio
    ref="media"
    @loadedmetadata="flowchartStore.playbackDuration = $event.target.duration"
    @timeupdate="flowchartStore.playbackPosition = $event.target.currentTime"
    @stalled="flowchartStore.mediaBuffering = true"
    @loadeddata="flowchartStore.mediaBuffering = false"
    @playing="flowchartStore.mediaBuffering = false"
    @ended="stopPlayback()"
  >
    <source
      src="/narration.mp3"
      type="audio/mp3"
      @error="flowchartStore.narrationEnabled = false"
    />
  </audio>
  -->
  <TheFlowchart
      v-if="flowchartStore.currentNarrationNodeId"
      @setCurrentNodeId="setCurrentNodeId"
      @jumpNarrationToNode="jumpNarrationToNode"
      @startPlayback="startPlayback"
      @startExplorationDuringPlayback="startExplorationDuringPlayback()"
      @stopExplorationDuringPlayback="stopExplorationDuringPlayback()"
      @toggleIntroPanel="toggleIntroPanel"
      @mousedown="toggleIntroPanel(true)"
  />
  <TheIntroPanel
      :siteTitle="userSetup.title"
      :siteSubtitle="userSetup.subtitle"
      @togglePlayback="togglePlayback()"
      @toggleIntroPanel="toggleIntroPanel()"
  />
  <ThePlaybackControls
      v-if="flowchartStore.narrationEnabled && flowchartStore.currentNarrationNodeId"
      :class="{ 'dark-scheme': this.darkScheme }"
      @jumpNarrationToNode="jumpNarrationToNode"
      @togglePlayback="togglePlayback()"
      @stopExplorationDuringPlayback="stopExplorationDuringPlayback()"
  />
</template>

<script>
import { mapStores, mapActions } from 'pinia';

import TheFlowchart from '@/components/TheFlowchart.vue';
import TheIntroPanel from '@/components/TheIntroPanel.vue';
import ThePlaybackControls from '@/components/ThePlaybackControls.vue';

import { useFlowchartStore } from '@/stores/FlowchartStore.js';
import { useViewStore } from '@/stores/ViewStore.js';

export default {
  name: 'App',

  components: {
    TheFlowchart,
    TheIntroPanel,
    ThePlaybackControls
  },

  data() {
    return {
      userSetup: {},
      darkScheme: false,

      // return to playback timeout reference and auto-return delay
      returnToPlaybackTimeout: undefined,
      returnToPlaybackDelay: 8000,

      // wakeLock support and reference
      wakeLockSupported: false,
      wakeLock: null
    }
  },

  computed: {
    ...mapStores(
        useFlowchartStore,
        useViewStore
    )
  },

  methods: {
    ...mapActions(useFlowchartStore, [
      'fetchTimestamps',
      'saveToLocalStorage',
      'resumeFromLocalStorage'
    ]),

    // fetch and apply user setup from public JSON file
    async fetchUserSetup() {
      fetch('setup.json')
          .then(response => response.json())
          .then(data => {
            this.userSetup = data;
            this.applyUserSetup();
          })
          .catch(error => console.log(error));
    },
    applyUserSetup() {
      this.flowchartStore.flowchartId = this.userSetup.id;

      // attempt to get state from previous session
      this.resumeFromLocalStorage();

      document.title = this.userSetup.title;

      for (const [property, color] of Object.entries(this.userSetup.colors)) {
        document.documentElement.style.setProperty('--' + property, this.hexToRgb(color));
      }

      // detect if background color is dark or light (relevant for icon colors)
      this.darkScheme = this.isDarkColor(this.userSetup.colors['background-color']);
    },

    // update the current node ID
    setCurrentNodeId(nodeId) {
      this.flowchartStore.currentNodeId = nodeId;
      this.saveToLocalStorage();
    },

    // jump playback position to node (either current node or node clicked during playback)
    jumpNarrationToNode(nodeId) {
      const nodeOccurrencesInNarration = this.flowchartStore.narrationTimestamps.filter(event => event[1] === nodeId);

      if (nodeOccurrencesInNarration.length !== 0) {
        // if playback is active, jump to the next occurrence after the current playback position or the final one if none exists
        if (this.flowchartStore.playbackActive) {
          const nextNodeOccurrenceAfterPlaybackPosition = nodeOccurrencesInNarration.find(event => event[0] >= this.flowchartStore.playbackPosition);

          if (nextNodeOccurrenceAfterPlaybackPosition) {
            this.setPlaybackPosition(nextNodeOccurrenceAfterPlaybackPosition[0]);
          } else {
            const lastNodeOccurrence = nodeOccurrencesInNarration[nodeOccurrencesInNarration.length - 1];
            this.setPlaybackPosition(lastNodeOccurrence[0]);
          }
          // if playback is paused, jump to the first unlistened occurrence in narration sequence or the final one if none exists
        } else {
          let furthestNodeOccurrence = [0, ''];

          const noUnlistenedOccurrences = nodeOccurrencesInNarration.every(event => {
            if (this.flowchartStore.listenedTimestampIndexes.indexOf(this.flowchartStore.narrationTimestamps.indexOf(event)) === -1) {
              this.setPlaybackPosition(event[0]);
              return false;
            } else {
              furthestNodeOccurrence = event;
              return true;
            }
          });

          if (noUnlistenedOccurrences) {
            this.setPlaybackPosition(furthestNodeOccurrence[0]);
          }
        }
      } else {
        // if destination node does not occur within narration (happens when jump is triggered through node’s
        // click event while playback is active), stop playback and update the current node ID instead
        this.stopPlayback();
        this.setCurrentNodeId(nodeId);
      }
    },

    // update current playback position and set currentTime of media to that position
    // (timeupdate event of media will keep updating playback position once ready/buffered)
    setPlaybackPosition(playbackPosition) {
      this.flowchartStore.playbackPosition = playbackPosition;
      this.$refs.media.currentTime = playbackPosition;
    },

    // start playback
    startPlayback(nodeIdAlreadySet = false) {
      if (!nodeIdAlreadySet) {
        this.setCurrentNodeId(this.flowchartStore.currentNarrationNodeId);
      }

      this.$refs.media.play();
      this.flowchartStore.playbackActive = true;
      this.requestWakeLock();
      this.toggleIntroPanel(true);
    },

    // stop playback
    stopPlayback() {
      this.$refs.media.pause();
      this.flowchartStore.playbackActive = false;
      this.releaseWakeLock();
      this.stopExplorationDuringPlayback();
    },

    // toggle playback
    togglePlayback() {
      this.flowchartStore.playbackActive ? this.stopPlayback() : this.startPlayback();
    },

    // start exploration during playback and set timeout to return to playback location automatically
    startExplorationDuringPlayback() {
      this.flowchartStore.exploringDuringPlayback = true;
      clearTimeout(this.returnToPlaybackTimeout);

      this.returnToPlaybackTimeout = setTimeout(this.stopExplorationDuringPlayback, this.returnToPlaybackDelay);
    },

    // stop exploration during playback
    stopExplorationDuringPlayback() {
      this.flowchartStore.exploringDuringPlayback = false;
      clearTimeout(this.returnToPlaybackTimeout);
    },

    // toggle visibility of intro panel
    toggleIntroPanel(forceClose = false) {
      if (forceClose || this.viewStore.introPanelVisible) {
        this.viewStore.introPanelVisible = false;
      } else {
        this.viewStore.introPanelVisible = true;
        this.stopPlayback();
      }
    },

    // request and release wakeLock if supported
    async requestWakeLock() {
      if (this.wakeLockSupported) {
        try {
          this.wakeLock = await navigator.wakeLock.request('screen');
        } catch {
          // nevermind
        }
      }
    },
    async releaseWakeLock() {
      if (this.wakeLockSupported && this.wakeLock) {
        this.wakeLock.release().then(() => {
          this.wakeLock = null;
        })
      }
    },

    isDarkColor(hex) {
      const rgb = this.hexToRgb(hex).split(',');
      const hsp = Math.sqrt(
          0.299 * (rgb[0] * rgb[0]) +
          0.587 * (rgb[1] * rgb[1]) +
          0.114 * (rgb[2] * rgb[2])
      );

      return hsp > 127.5 ? false : true;
    },

    hexToRgb(hex) {
      const shorthandRegex = /^#?([a-f\d])([a-f\d])([a-f\d])$/i;
      hex = hex.replace(shorthandRegex, function(m, r, g, b) {
        return r + r + g + g + b + b;
      });

      const result = /^#?([a-f\d]{2})([a-f\d]{2})([a-f\d]{2})$/i.exec(hex);
      return result ? parseInt(result[1], 16) + ',' + parseInt(result[2], 16) + ',' + parseInt(result[3], 16) : null;
    }
  },

  created() {
    this.fetchUserSetup();

    //toDo: remove when adding narration.mp3 again
    this.flowchartStore.narrationEnabled = false;

    // set wakeLockSupported if supported
    if ('wakeLock' in navigator) {
      this.wakeLockSupported = true;
    }

    // fetch timestamps from public JSON
    this.fetchTimestamps();
  },

  mounted() {
    // if resumed from storage, set currentTime of media to playbackPosition from storage
    if (this.flowchartStore.resumedFromLocalStorage) {
      this.setPlaybackPosition(this.flowchartStore.playbackPosition);
    }

    // make spacebar trigger togglePlayback
    document.addEventListener('keydown', event => {
      if (event.key === ' ' || event.key === 'Space') {
        event.preventDefault();
        this.togglePlayback();
      } else if (event.key === 'Enter' && this.flowchartStore.jumpActionAvailable) {
        event.preventDefault();
        this.jumpNarrationToNode(this.flowchartStore.currentNodeId);
      }
    });

    document.addEventListener('touchstart', () => {}, true);
  }
}
</script>

<style lang="scss">
@import '@/assets/normalize.css';
@import '@/assets/fonts.css';
@import '@/assets/variables.css';

body {
  width: 100%;
  height: 100%;
  overflow: hidden;
  background: rgb(var(--background-color));
}

#app {
  display: contents;
  font-family: 'IBM Plex Sans', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen-Sans, Ubuntu, Cantarell, 'Helvetica Neue', sans-serif;
  font-size: 15px;
  text-rendering: optimizeLegibility;
  color: rgb(var(--text-color));
  -webkit-tap-highlight-color: rgba(0,0,0,0);
}

*:focus {
  outline: none;
}

*:focus-visible {
  box-shadow: 0 0 0 2px rgb(var(--background-color)), 0 0 0 4px rgb(var(--text-color)) !important;
}
</style>
