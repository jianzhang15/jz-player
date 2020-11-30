<template>
  <div
    class="aplayer"
    :class="{
      'aplayer-mini': mini,
      'aplayer-withlist' : !mini && musicList.length > 0,
      'aplayer-withlrc': !mini && (!!$slots.display || showLrc),
      'aplayer-float': isFloatMode,
      'aplayer-loading': isPlaying && isLoading,
      'isCollpase':isTogglePanel
    }"
    :style="floatStyleObj"
  >
    <div class="aplayer-body">
      <thumbnail
        :pic="currentMusic.pic"
        :playing="isPlaying"
        :enable-drag="isFloatMode"
        :theme="currentTheme"
        @toggleplay="toggle"
        @dragbegin="onDragBegin"
        @dragging="onDragAround"
      />
      <div class="aplayer-info" :class="{'isCollpase':isTogglePanel}" v-show="!mini">
        <div class="aplayer-music">
          <!-- <span class="aplayer-title">{{ currentMusic.title || 'Untitled' }}</span>
          <span class="aplayer-author">{{ currentMusic.artist || 'Unknown' }}</span> -->
          <div id="wavesurfer" style="width:400px"></div>
        </div>
        <slot name="display" :current-music="currentMusic" :play-stat="playStat">
          <lyrics :current-music="currentMusic" :play-stat="playStat" v-if="showLrc" />
        </slot>
        <controls
          :shuffle="shouldShuffle"
          :repeat="repeatMode"
          :stat="playStat"
          :volume="audioVolume"
          :muted="isAudioMuted"
          :theme="currentTheme"
          @toggleshuffle="shouldShuffle = !shouldShuffle"
          @togglelist="showList = !showList"
          @togglemute="toggleMute"
          @setvolume="setAudioVolume"
          @dragbegin="onProgressDragBegin"
          @dragend="onProgressDragEnd"
          @dragging="onProgressDragging"
          @nextmode="setNextMode"
        />
      </div>
      <div class="collpaseBtn" @click="togglePanel">
        <img v-if="!isTogglePanel" src="./assets/close.png" style="width: 20px;"/>
        <img v-else src="./assets/open.png" style="width: 20px;"/>
      </div>
    </div>
    <music-list
      :show="showList && !mini"
      :current-music="currentMusic"
      :music-list="musicList"
      :play-index="playIndex"
      :listmaxheight="listMaxHeight"
      :theme="currentTheme"
      @selectsong="onSelectSong"
    />
    <loading v-if="waveLoading"></loading>
  </div>
</template>
<script type="text/babel">
  import Vue from 'vue'
  import Thumbnail from './components/aplayer-thumbnail.vue'
  import MusicList from './components/aplayer-list.vue'
  import Controls from './components/aplayer-controller.vue'
  import Lyrics from './components/aplayer-lrc.vue'
  import wavesurfer from "wavesurfer.js";
  import loading from './components/loading/loading'
  Vue.use(loading)
  import { deprecatedProp, versionCompare, warn } from './utils'
  import Icon from './components/aplayer-icon.vue'
  let versionBadgePrinted = false
  const canUseSync = versionCompare(Vue.version, '2.3.0') >= 0

  /**
   * memorize self-adapting theme for cover image urls
   * @type {Object.<url, rgb()>}
   */
  const picThemeCache = {}

  // mutex playing instance
  let activeMutex = null


  const REPEAT = {
    NONE: 'none',
    MUSIC: 'music',
    LIST: 'list',
    NO_REPEAT: 'no-repeat',
    REPEAT_ONE: 'repeat-one',
    REPEAT_ONCE: 'repeat-once',
    REPEAT_ALL: 'repeat-all',
  };

  const VueAPlayer = {
    name: 'APlayer',
    disableVersionBadge: false,
    components: {
      Thumbnail,
      Controls,
      MusicList,
      Lyrics,
      Icon
    },
    props: {
      music: {
        type: Object,
        required: true,
        validator (song) {
          return !!song.src
        },
      },
      list: {
        type: Array,
        default () {
          return []
        },
      },
      mini: {
        type: Boolean,
        default: false,
      },
      showLrc: {
        type: Boolean,
        default: false,
      },
      mutex: {
        type: Boolean,
        default: true,
      },
      theme: {
        type: String,
        default: '#647ccd',
      },

      listMaxHeight: String,
      /**
       * @since 1.4.1
       * Fold playlist initially
       */
      listFolded: {
        type: Boolean,
        default: false,
      },

      /**
       * @since 1.2.0 Float mode
       */
      float: {
        type: Boolean,
        default: false,
      },

      // Audio attributes as props
      // since 1.4.0
      // autoplay controls muted preload volume
      // autoplay is not observable

      /**
       * @since 1.4.0
       * not observable
       */
      autoplay: {
        type: Boolean,
        default: false,
      },

      /**
       * @since 1.4.0
       * whether to show native audio controls below Vue-APlayer
       * only work in development environment and not mini mode
       *
       * observable
       */
      controls: {
        type: Boolean,
        default: false,
      },

      /**
       * @since 1.4.0
       * observable, sync
       */
      muted: {
        type: Boolean,
        default: false,
      },
      /**
       * @since 1.4.0
       * observable
       */
      preload: String,

      /**
       * @since 1.4.0
       * observable, sync
       */
      volume: {
        type: Number,
        default: 1,
        validator (value) {
          return value >= 0 && value <= 1
        },
      },

      // play order control
      // since 1.5.0

      /**
       * @since 1.5.0
       * @see https://support.apple.com/en-us/HT207230
       * twoWay
       */
      shuffle: {
        type: Boolean,
        default: false,
      },
      /**
       * @since 1.5.0
       * @see https://support.apple.com/en-us/HT207230
       * twoWay
       */
      repeat: {
        type: String,
        default: "REPEAT.NO_REPEAT",
      },
    },
    data () {
      return {
        internalMusic: this.music,
        isPlaying: false,
        isSeeking: false,
        wasPlayingBeforeSeeking: false,
        isMobile: /mobile/i.test(window.navigator.userAgent),
        playStat: {
          duration: 0,
          loadedTime: 0,
          playedTime: 0,
        },
        showList: !this.listFolded,
        // @since 1.2.0 float mode
        floatOriginX: 0,
        floatOriginY: 0,
        floatOffsetLeft: 0,
        floatOffsetTop: 0,


        // @since 1.3.0 self adapting theme
        selfAdaptingTheme: null,


        // @since 1.4.0
        // sync muted, volume

        internalMuted: this.muted,
        internalVolume: this.volume,

        // @since 1.4.1
        // Loading indicator
        isLoading: false,


        // @since 1.5.1
        // sync shuffle, repeat
        internalShuffle: this.shuffle,
        internalRepeat: this.repeat,
        // for shuffling
        shuffledList: [],
        wavesurfer: null,
        options: {
          container: "#wavesurfer",
          cursorColor: "#F7B500",
          cursorWidth: 2,
          barWidth: 1.5,
          barHeight: 1, // the height of the wave
          height: 30,
          waveColor: "#9BAADB",
          progressColor: "#4B74FF",
          backgroundColor: "#f8f8f8",
          loopSelection: true,
          scrollParent: false,
          normalize: true,
        },
        waveLoading:false,
        isTogglePanel:false,
        afterLoadPlay:true
      }
    },
    computed: {
      // sync music
      currentMusic: {
        get () {
          return this.internalMusic
        },
        set (val) {
          canUseSync && this.$emit('update:music', val)
          this.internalMusic = val
        },
      },

      // props wrappers

      currentTheme () {
        return this.selfAdaptingTheme || this.currentMusic.theme || this.theme
      },
      isFloatMode () {
        return this.float && !this.isMobile
      },
      shouldAutoplay () {
        if (this.isMobile) return false
        return this.autoplay
      },
      musicList () {
        return this.list
      },
      shouldShowNativeControls () {
        return process.env.NODE_ENV !== 'production' &&
          this.controls &&
          !this.mini
      },

      // useful

      floatStyleObj () {
        // transform: translate(floatOffsetLeft, floatOffsetY)
        return {
          transform: `translate(${this.floatOffsetLeft}px, ${this.floatOffsetTop}px)`,
          webkitTransform: `translate(${this.floatOffsetLeft}px, ${this.floatOffsetTop}px)`,
        }
      },
      currentPicStyleObj () {
        if (this.currentMusic && this.currentMusic.pic) {
          return {
            backgroundImage: `url(${this.currentMusic.pic})`,
          }
        }
        return {}
      },
      loadProgress () {
        if (this.playStat.duration === 0) return 0
        return this.playStat.loadedTime / this.playStat.duration
      },
      playProgress () {
        if (this.playStat.duration === 0) return 0
        return this.playStat.playedTime / this.playStat.duration
      },
      playIndex: {
        get () {
          return this.shuffledList.indexOf(this.currentMusic)
        },
        set (val) {
          this.currentMusic = this.shuffledList[val % this.shuffledList.length]
        },
      },
      shouldRepeat () {
        return this.repeatMode !== REPEAT.NO_REPEAT
      },

      // since 1.4.0
      // sync muted, volume

      isAudioMuted: {
        get () {
          return this.internalMuted
        },
        set (val) {
          canUseSync && this.$emit('update:muted', val)
          this.internalMuted = val
        },
      },
      audioVolume: {
        get () {
          return this.internalVolume
        },
        set (val) {
          canUseSync && this.$emit('update:volume', val)
          this.internalVolume = val
        },
      },


      // since 1.5.0
      // sync shuffle, repeat
      shouldShuffle: {
        get () {
          return this.internalShuffle
        },
        set (val) {
          canUseSync && this.$emit('update:shuffle', val)
          this.internalShuffle = val
        },
      },
      repeatMode: {
        get () {
          switch (this.internalRepeat) {
            case "REPEAT.NONE":
            case "REPEAT.NO_REPEAT":
              return REPEAT.NO_REPEAT
              break;
            case "REPEAT.MUSIC":
            case "REPEAT.REPEAT_ONE":
              return REPEAT.REPEAT_ONE
              break;
            case "REPEAT.REPEAT_ONCE":
              return REPEAT.REPEAT_ONCE
              break;
            default:
              return REPEAT.REPEAT_ALL
              break;
          }
        },
        set (val) {
          canUseSync && this.$emit('update:repeat', val)
          this.internalRepeat = val
        },
      },
    },
    methods: {
      loadWave (){
        this.isTogglePanel=false
        return new Promise((resolve,reject)=>{
            if(this.wavesurfer){
              this.wavesurfer.stop()
              document.querySelector("#wavesurfer").childNodes.forEach(item=>{
                item.remove()
              })
              this.wavesurfer=null
            }
            this.waveLoading=true
            this.wavesurfer = wavesurfer.create(this.options)
            this.wavesurfer.load(this.internalMusic.src);
            this.wavesurfer.on("ready", () => {
              this.waveLoading=false
              this.onAudioCanplay()
              this.play()
              if(!this.afterLoadPlay) this.pause()
              resolve()
            });
          })
      },
      // Float mode

      onDragBegin () {
        this.floatOriginX = this.floatOffsetLeft
        this.floatOriginY = this.floatOffsetTop
      },
      onDragAround ({ offsetLeft, offsetTop }) {
        this.floatOffsetLeft = this.floatOriginX + offsetLeft
        this.floatOffsetTop = this.floatOriginY + offsetTop
      },

      // functions

      setNextMode () {
        if (this.repeatMode === REPEAT.REPEAT_ALL) {
          this.repeatMode = REPEAT.REPEAT_ONE
        } else if (this.repeatMode === REPEAT.REPEAT_ONE) {
          this.repeatMode = REPEAT.NO_REPEAT
        }else if (this.repeatMode === REPEAT.REPEAT_ONCE) {
          this.repeatMode = REPEAT.REPEAT_ONCE
        } else {
          this.repeatMode = REPEAT.REPEAT_ALL
        }
      },
      thenPlay () {
        this.$nextTick(() => {
          this.play()
        })
      },

      // controls

      // play/pause

      toggle () {
        if (this.isPlaying) {
          this.pause()
        } else {
          this.play()
        }
      },
      play () {
        this.$emit("play",this.currentMusic)
        // this.isTogglePanel=false
        this.isPlaying=true
        this.wavesurfer.play()
      },
      pause () {
        this.$emit("pause",this.currentMusic)
        // this.isTogglePanel=false
        this.isPlaying=false
        this.wavesurfer.pause()
      },
      stop () {
        this.$emit("stop",this.currentMusic)
        // this.isTogglePanel=false
        this.isPlaying=false
        this.wavesurfer.stop()
      },

      // progress bar

      onProgressDragBegin (val) {
        this.wasPlayingBeforeSeeking = this.isPlaying
        this.pause()
        this.isSeeking = true
        this.wavesurfer.seekTo(val);
      },
      onProgressDragging (val) {
        this.wavesurfer.seekTo(val);
      },
      onProgressDragEnd (val) {
        this.isSeeking = false

        if (this.wasPlayingBeforeSeeking) {
          this.thenPlay()
        }
      },

      // volume

      toggleMute (event) {
        this.setAudioMuted(!this.wavesurfer.getMute())
      },
      setAudioMuted (val) {
        this.wavesurfer.setMute(val)
      },
      setAudioVolume (val) {
        this.wavesurfer.setVolume(val)
        if (val > 0) {
          this.setAudioMuted(false)
        }
      },

      // playlist

      getShuffledList () {
        if (!this.list.length) {
          return [this.internalMusic]
        }
        let unshuffled = [...this.list]
        if (!this.internalShuffle || unshuffled.length <= 1) {
          return unshuffled
        }

        let indexOfCurrentMusic = unshuffled.indexOf(this.internalMusic)
        if (unshuffled.length === 2 && indexOfCurrentMusic !== -1) {
          if (indexOfCurrentMusic === 0) {
            return unshuffled
          } else {
            return [this.internalMusic, unshuffled[0]]
          }
        }
        // shuffle list
        // @see https://stackoverflow.com/questions/6274339/how-can-i-shuffle-an-array
        for (let i = unshuffled.length - 1; i > 0; i--) {
          const j = Math.floor(Math.random() * (i + 1))
          const tmp = unshuffled[i]
          unshuffled[i] = unshuffled[j]
          unshuffled[j] = tmp
        }

        // take currentMusic to first
        if (indexOfCurrentMusic !== -1) {
          indexOfCurrentMusic = unshuffled.indexOf(this.internalMusic)
          if (indexOfCurrentMusic !== 0) {
            [unshuffled[0], unshuffled[indexOfCurrentMusic]] = [unshuffled[indexOfCurrentMusic], unshuffled[0]]
          }
        }

        return unshuffled
      },

      onSelectSong (song) {
        if (this.currentMusic === song) {
          this.toggle()
        } else {
          this.afterLoadPlay=true
          this.currentMusic = song
          // this.thenPlay()
        }
      },

      // event handlers
      // for keeping up with audio states

      onAudioPlay () {
        this.isPlaying = true
      },
      onAudioPause () {
        this.isPlaying = false
      },
      onAudioCanplay () {
        this.initAudio()
        this.playStat.duration = this.wavesurfer.getDuration()
        this.playStat.loadedTime = this.wavesurfer.getDuration()
        this.playStat.playedTime=0;
        this.isLoading = false
      },
      onAudioDurationChange () {
        if (this.wavesurfer.getDuration() !== 1) {
          this.playStat.duration = this.wavesurfer.getDuration()
        }
      },
      onAudioProgress () {
        this.playStat.loadedTime = this.wavesurfer.getDuration()
      },
      onAudioTimeUpdate () {
        this.playStat.playedTime = this.wavesurfer.getCurrentTime()
      },
      onAudioSeeking () {
        this.playStat.playedTime = this.wavesurfer.getCurrentTime()
        if(this.isPlaying) this.thenPlay()
      },
      onAudioSeeked () {
        this.playStat.playedTime = this.wavesurfer.getCurrentTime()
      },
      onAudioVolumeChange (e) {
        this.$nextTick(_=>{
          this.audioVolume = this.wavesurfer.getVolume()
          this.isAudioMuted = this.wavesurfer.getMute()
        })
      },
      onAudioEnded () {
        // determine next song according to shuffle and repeat
        if (this.repeatMode === REPEAT.REPEAT_ALL) {
          if (this.shouldShuffle && this.playIndex === this.shuffledList.length - 1) {
            this.shuffledList = this.getShuffledList()
          }
          this.playIndex++
          this.afterLoadPlay=true
        } else if (this.repeatMode === REPEAT.REPEAT_ONE) {
          this.wavesurfer.seekTo(0)
        } else if (this.repeatMode === REPEAT.REPEAT_ONCE) {
          if (this.shouldShuffle && this.playIndex === this.shuffledList.length - 1) {
            this.shuffledList = this.getShuffledList()
          }
          this.pause()
          this.wavesurfer.seekTo(0)
          this.playIndex++
          this.afterLoadPlay=false
        }else if(this.repeatMode === REPEAT.NO_REPEAT){
          this.$emit("end",this.currentMusic)
          this.pause()
          this.wavesurfer.seekTo(0)
        }else {
          this.playIndex++
          if (this.playIndex !== 0) {
            this.afterLoadPlay=true
          } else if (this.shuffledList.length === 1) {
            this.wavesurfer.seekTo(0)
          }else{
            this.afterLoadPlay=false
          }
        }
      },

      initAudio () {
        // event handlers
        // they don't emit native media events
        this.wavesurfer.on('seek',this.onAudioSeeking)
        this.wavesurfer.on('audioprocess',this.onAudioTimeUpdate)
        this.wavesurfer.on('volume',this.onAudioVolumeChange)
        this.wavesurfer.on('finish',this.onAudioEnded)
      },

      setSelfAdaptingTheme () {
        // auto theme according to current music cover image
        if ((this.currentMusic.theme || this.theme) === 'pic') {
          const pic = this.currentMusic.pic
          // use cache
          if (picThemeCache[pic]) {
            this.selfAdaptingTheme = picThemeCache[pic]
          } else {
            try {
              new ColorThief().getColorAsync(pic, ([r, g, b]) => {
                picThemeCache[pic] = `rgb(${r}, ${g}, ${b})`
                this.selfAdaptingTheme = `rgb(${r}, ${g}, ${b})`
              })
            } catch (e) {
              warn('color-thief is required to support self-adapting theme')
            }
          }
        } else {
          this.selfAdaptingTheme = null
        }
      },
      togglePanel (){
        // if(this.isTogglePanel===false) this.showList=false;
        this.isTogglePanel=!this.isTogglePanel
      }
    },
    watch: {
      music (music) {
        this.internalMusic = music
      },
      list (list){
        this.shuffledList = this.getShuffledList()
        this.afterLoadPlay=true
      },
      currentMusic: {
        async handler (music) {
          // async
          await this.loadWave()
          this.setSelfAdaptingTheme()
          if(this.playIndex!=0) this.thenPlay()
          // self-adapting theme color
        },
      },

      // since 1.4.0
      // observe controls, muted, preload, volume

      // shouldShowNativeControls (val) {
      //   this.audio.controls = val
      // },
      isAudioMuted (val) {
        this.wavesurfer.setMute(val)
      },
      // preload (val) {
      //   this.audio.preload = val
      // },
      audioVolume (val) {
        this.wavesurfer.setVolume(val)
      },

      // sync muted, volume

      muted (val) {
        this.internalMuted = val
      },
      volume (val) {
        this.internalVolume = val
      },


      // sync shuffle, repeat
      shuffle (val) {
        this.internalShuffle = val
      },
      repeat (val) {
        this.internalRepeat = val
      },
    },
    beforeCreate () {
      if (!VueAPlayer.disableVersionBadge && !versionBadgePrinted) {
        // version badge
        console.log(`\n\n %c 国音Speakin音乐播放器${VERSION} %c\n`, 'color: #fff; background:#41b883; padding:5px 0;', 'color: #fff; background: #35495e; padding:5px 0;')
        versionBadgePrinted = true
      }
    },
    created () {
      this.shuffledList = this.getShuffledList()
    },
    async mounted () {
      this.afterLoadPlay=this.autoplay
      await this.loadWave()
      this.setSelfAdaptingTheme()
      if (this.autoplay) this.play()
    },
    beforeDestroy () {
      if (activeMutex === this) {
        activeMutex = null
      }
      // if (this.hls) {
      //   this.hls.destroy()
      // }
    },
  }

  export default VueAPlayer

</script>

<style lang="scss">
  @import "./scss/variables";

  .aplayer {
    font-family: Arial, Helvetica, sans-serif;
    color: #000;
    background-color: #fff;
    margin: 5px;
    box-shadow: 0 2px 2px 0 rgba(0, 0, 0, 0.07), 0 1px 5px 0 rgba(0, 0, 0, 0.1);
    border-radius: 2px;
    overflow: hidden;
    user-select: none;
    line-height: initial;
    transition: all 0.2s;
    &.isCollpase{
      width: 90px;
    }
    * {
      box-sizing: content-box;
    }

    .aplayer-lrc-content {
      display: none;
    }

    .aplayer-body {
      display: flex;

      position: relative;
      .aplayer-info {
        &.isCollpase{
          display: none;
        }
        flex-grow: 1;
        display: flex;
        flex-direction: column;

        text-align: start;
        padding: 14px 7px 0 10px;
        height: $aplayer-height;
        box-sizing: border-box;
        overflow: hidden;

        .aplayer-music {
          flex-grow: 1;

          overflow: hidden;
          white-space: nowrap;
          text-overflow: ellipsis;
          margin-left: 5px;
          user-select: text;
          cursor: default;
          padding-bottom: 2px;

          .aplayer-title {
            font-size: 14px;
          }

          .aplayer-author {
            font-size: 12px;
            color: #666;
          }
        }

        .aplayer-lrc {
          z-index: 0;
        }
      }
    }
    // Mini mode
    &.aplayer-mini {
      width: $aplayer-height;
    }

    &.aplayer-withlrc {
      .aplayer-body {
        .aplayer-pic {
          height: $aplayer-height-lrc;
          width: $aplayer-height-lrc;
        }

        .aplayer-info {
          height: $aplayer-height-lrc;
        }

        .aplayer-info {
          padding: 10px 7px 0 7px;
        }
      }
    }

    &.aplayer-withlist {
      .aplayer-body {
        .aplayer-info {
          border-bottom: 1px solid #e9e9e9;
        }

        .aplayer-controller .aplayer-time .aplayer-icon.aplayer-icon-menu {
          display: block;
        }
      }
    }

    /* floating player on top */
    position: fixed;
    width: 500px;
    bottom: 20px;
    right: 20px;
    z-index:99999
  }

  @keyframes aplayer-roll {
    0% {
      left: 0
    }
    100% {
      left: -100%
    }
  }
  .collpaseBtn{
    padding: 0 3px;
    display: flex;
    align-items: center;
    cursor: pointer;
  }
</style>