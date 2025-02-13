<template>
  <div id="fullscreen-wrapper" @click="(e) => $_exitFullscreen(e)" :class="isFullScreen ? 'fullscreen-video' : 'video-wrapper'">
    <div ref="VideoWrapper" class="video-container">
      <!-- VISUAL DISPLAY -->
      <div ref="CanvasWrapper" style="position: relative;">
        <v-overlay :value="!isPlaying"
          absolute
          color="white"
          :opacity="0.1"
        >
          <v-progress-circular v-if="isLoading"
            indeterminate
            color="amber"
            size="50"
            :width="5"
          ></v-progress-circular>
          
          <v-btn v-else 
            @click="playAudio()"
            x-large
            class="mx-2"
            fab
            dark
            color="white"
          >
            <v-icon dark style="font-size: 3.5rem; opacity: 40%" color="black">
              {{ isPlaying ? 'mdi-pause' : 'mdi-play' }}
            </v-icon>
          </v-btn>
        </v-overlay>

        <canvas @click="pauseAudio()"
          ref="FrontCanvas" 
          class="front-canvas"
        >
        </canvas>

        <canvas 
          ref="BackCanvas" 
          class="background-canvas"
        >
        </canvas>
      </div>

      <!-- AUDIO SLIDER -->
      <audio v-if="audioUrl && strokesArray.length > 0"
        :src="audioUrl" 
        @play="initSyncing(); $emit('play'); isPlaying = true"
        @pause="isPlaying = false"
        @ended="isPlaying = false"
        @seeking="syncStrokesToAudio()"
        ref="AudioPlayer" 
        style="width: 100%;"
        controls
      />
      <div class="extra-controls">
        <!-- <v-btn @click.stop="$_toggleFullscreen()">
          <v-icon>mdi-fullscreen</v-icon>
        </v-btn> -->
        <v-menu>
          <template v-slot:activator="{ on }">
            <v-btn v-on="on" @click.stop dark color="grey darken-1">
              <v-icon color="white">mdi-dots-vertical</v-icon>
            </v-btn>
          </template>

          <v-list>
            <v-list-item style="display: flex">
              <v-btn v-for="speedOption of speedOptions" :key="speedOption.value" text outlined
                @click="changePlaybackSpeed(speedOption.value)"
              >
                {{ speedOption.text }}
              </v-btn>
            </v-list-item>

            <v-list-item @click.stop="$emit('delete')">
              <v-icon class="mr-2">mdi-delete</v-icon> REVERT TO BLACKBOARD
            </v-list-item>
          </v-list>
        </v-menu>
      </div>
      <!-- End of #extra-controls -->
    </div>
  </div>
</template>

<script>
import DoodleFullscreenMixin from "@/mixins/DoodleFullscreenMixin.js";
import CanvasDrawMixin from "@/mixins/CanvasDrawMixin.js";
import _ from "lodash";
import { 
  PPT_SLIDE_RATIO,
} from "@/CONSTANTS.js";

export default {
  props: {
    strokesArray: {
      type: Array,
      required: true
    },
    audioUrl: {
      type: String,
      required: true
    },
    imageBlob: Blob, // a File is also a Blob,
    aspectRatio: {
      type: Number,
      default () {
        return PPT_SLIDE_RATIO; 
      }
    },
    isLoading: {
      type: Boolean,
      required: true
    }
  },
  mixins: [
    CanvasDrawMixin,
    DoodleFullscreenMixin
  ],
  data: () => ({
    playbackSpeed: 1,
    speedOptions: [{text:'0.5x', value: 0.5},{text:'1x', value: 1},{text:'1.5x', value: 1.5},{text:'2x', value: 2},{text:'3x', value: 3}],
    canvas: null,
    ctx: null,
    bgCanvas: null,
    bgCtx: null,
    nextFrameIdx: 0, 
    recursiveSyncer: null,
    isPlaying: false
  }),
  computed: {
    imageBlobUrl () {
      if (this.imageBlob) {
        return URL.createObjectURL(this.imageBlob);
      } else {
        return "";
      }
    },
    /* Converts separate strokes into continuous points */
    allFrames () {
      const allPoints = [];
      for (let i = 0; i < this.strokesArray.length; i++) {
        for (let j = 1; j < this.strokesArray[i].points.length; j++) {
          const frame = { strokeIndex: i, pointIndex: j };
          allPoints.push({ 
            startTime: this.getStartTime(frame),
            ...frame, 
          }); 
        }
      }
      return allPoints.sort((p1, p2) => p1.startTime - p2.startTime);
    }
  },
  watch: {
    // quick-fix, without it the video appears blank even if `strokesArray` changed from empty to hydrated
    // assumptions:
    //   1. `strokesArray` only changes at most ONCE
    //   2. `imageBlobUrl` is hydrated at the SAME time as `strokesArray`
    strokesArray () {
      this.$_drawStrokesInstantly()
      this.$_renderBackground(this.imageBlobUrl)
    }
  },
  async mounted () {
    this.canvas = this.$refs.FrontCanvas;
    this.bgCanvas = this.$refs.BackCanvas;
    this.ctx = this.canvas.getContext("2d");
    this.bgCtx = this.bgCanvas.getContext("2d");
    await this.handleResize();
    this.handleResize = _.debounce(this.handleResize, 100); 
    window.addEventListener("resize", this.handleResize);

    // quickfix for 2x speed
    // await this.$nextTick(); 
    // await this.$nextTick()
    // this.changePlaybackSpeed(this.playbackSpeed);
  },
  beforeDestroy () {
    // quickfix for the previous recordings playing bug on iOS
    const { AudioPlayer } = this.$refs;
    if (AudioPlayer) AudioPlayer.src = '';
    
    window.removeEventListener("resize", this.handleResize);

    // clean up the recursive syncing
    clearTimeout(this.recursiveSyncer);
  },
  methods: {
    changePlaybackSpeed (speed) {
      this.playbackSpeed = speed
      this.$refs.AudioPlayer.playbackRate = speed;
    },
    playAudio () {
      this.$refs.AudioPlayer.play();
    },
    pauseAudio () {
      this.$refs.AudioPlayer.pause();
    },
    // TODO: touch to play or pause
    getStartTime ({ strokeIndex, pointIndex }) {
      const stroke = this.strokesArray[strokeIndex];
      return stroke.startTime + (pointIndex - 1) * this.$_getPointDuration(stroke);
    },
    handleResize () {
      return new Promise(async (resolve) => {
        this.resizeVideo();
        this.$_rescaleCanvas();
        if (this.recursiveSyncer) {
          // video was playing: resume to previous progress
          this.nextFrameIdx = 0;
          this.syncStrokesToAudio();
        } else {
          this.$_drawStrokesInstantly();
        }
        resolve();
      })
    },
    resizeVideo () {
      // TODO: what's the difference between CanvasWrapper and VideoWrapper?
      const { CanvasWrapper, VideoWrapper } = this.$refs;
      VideoWrapper.style.width = "100%";
    
      const availableWidth = VideoWrapper.offsetWidth
  
      const heightOfGapToShowGlimpseOfNextBoard = 30 // the distance between blackboards is half i.e. 10
      const availableHeight = window.innerHeight - heightOfGapToShowGlimpseOfNextBoard
      let videoHeight; 
      let videoWidth; 

      if (availableWidth * (1/this.aspectRatio) < availableHeight) {
        videoWidth = availableWidth;
        videoHeight = videoWidth * (1/this.aspectRatio);
      } else {
        videoHeight = availableHeight;
        videoWidth = videoHeight * this.aspectRatio;
      }

      CanvasWrapper.style.height = `${videoHeight}px`;
      this.canvas.style.height = `${videoHeight}px`;
      this.bgCanvas.style.height = `${videoHeight}px`;
      VideoWrapper.style.width = `${videoWidth}px`;
    },
    initSyncing () {
      this.nextFrameIdx = 0;
      this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height); // video could already be rendered as an initial preview or completed video
      this.syncRecursively();
    },
    syncRecursively () {
      const { AudioPlayer } = this.$refs;
      if (!AudioPlayer) return;
      this.syncStrokesToAudio();
      if (this.nextFrameIdx < this.allFrames.length) {
        // calculate sleep duration
        const nextFrame = this.allFrames[this.nextFrameIdx];
        const timeTilNextStroke = 1000 * (nextFrame.startTime - AudioPlayer.currentTime); 
        this.recursiveSyncer = setTimeout(this.syncRecursively, timeTilNextStroke/this.playbackSpeed); // use recursion instead of `setInterval` to prevent overlapping calls
      }
    },
    syncStrokesToAudio () {
      const { AudioPlayer } = this.$refs;
      if (!AudioPlayer) return;
      const nextFrame = this.allFrames[this.nextFrameIdx];
      if (!nextFrame || nextFrame.startTime > AudioPlayer.currentTime) { // !nextFrame: nextFrame is undefined after a video finishes
        this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);
        this.nextFrameIdx = 0;
      }
      this.renderFramesUntilCurrentTime();
    },
    renderFramesUntilCurrentTime () {
      const { currentTime } = this.$refs.AudioPlayer;
      for (let i = this.nextFrameIdx; i < this.allFrames.length; i++) {
        const frame = this.allFrames[i];
        if (frame.startTime > currentTime) { 
          break; 
        }
        this.renderFrame(frame);
        this.nextFrameIdx += 1;
      }
    },
    renderFrame ({ strokeIndex, pointIndex }) {
      const stroke = this.strokesArray[strokeIndex];
      const lineWidth = stroke.lineWidth; 
      const normalizedLineWidth = lineWidth * (this.canvas.scrollWidth / 1000);
      this.$_connectTwoPoints(
        stroke.points, 
        pointIndex, 
        stroke.isErasing, 
        this.ctx,
        stroke.color,
        normalizedLineWidth
      );
    }
  }
}
</script>

<style scoped>
@import "../styles/doodle-fullscreen.scss";

.video-container {
  position: relative;
}
.doodle-video {
  height: 100%; 
  width: 100%;
  position: relative; 
  z-index: 5; 
  margin: auto;
}
.canvas-wrapper {
  width: 100%;
  height: 100%; 
  position: relative;
}
.front-canvas {
  width: 100%; 
  height: 1; 
  background-color: transparent; 
  display: block;
}
.background-canvas {
  /* absolute places the background relative to the parent and ignore the front canvas, thereby stacking on top of each other */
  position: absolute; 
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: -1;
  background-color: rgb(46, 49, 49);
}
.extra-controls {
  position: absolute;
  top: 10px;
  left: 10px;
  width: calc(100% - 20px);
  display: flex;
  justify-content: flex-end;
  z-index: 1000
}
.extra-controls > * {
  margin: 0 5px;
}
.extra-controls > *:hover {
  opacity: 1;
}
</style>
<style>
.extra-controls .v-select__selections {
  color: accent;
}
</style>