<template>
  <div>
    <!-- Camera/Mic permission errors -->
    <VideoTroubleshootPopup v-model="isShowingVideoTroubleshootPopup"/> 

    <VideoConferenceRoomGeneralErrorPopup v-model="isShowingGeneralErrorPopup"
      :feedbackForUser="feedbackForUser"
    /> 

    <div id="container-for-audio-elements">

    </div>

    <portal to="current-room-participants">
      <portal-target name="video-screenshare-hangup-buttons">

      </portal-target>

      <template v-for="client in allClients">
        <!-- TODO: rename "id" -->
        <div 
          :id="client.sessionID === sessionID ? 'my-local-video' : client.sessionID" 
          :key="client.id" 
          style="position: relative; height: max-content" 
          class="mt-1"
        >
          <div style="display: flex; justify-content: space-between; align-items: center; height: 30px; padding-left: 4px; padding-right: 0;"
            :class="participants.hasOwnProperty( firestoreIDToDailyID[client.sessionID] ) ? 
              (participants[firestoreIDToDailyID[client.sessionID]].video ? 'video-overlay' : '') : ''
              ||
              participants.hasOwnProperty('local') && client.sessionID === sessionID ? (participants.local.video ? 'video-overlay' : '') : ''
            "
          >
            <div
              style="font-weight: 400; font-size: 1em;"
              class="text--secondary d-flex caption"
            >
              <div :class="activeSpeakerDailyID === firestoreIDToDailyID[client.sessionID] ? 
                `caption font-weight-bold text--darken-3 grey--text` : 'text--secondary'" 
                style="align-items: center; display: flex; margin-left: 8px;"
              >
                <v-icon v-if="client.kind === 'engineer'" 
                  x-small style="opacity: 70%; margin-top: 2px;"
                >
                  mdi-wrench
                </v-icon>
                <v-icon v-else-if="client.kind === 'pirate'" 
                  x-small style="opacity: 70%; margin-top: 2px;"
                >
                  mdi-skull-crossbones
                </v-icon>
                <v-icon v-else-if="client.kind === 'pioneer'" 
                   x-small style="opacity: 70%; margin-top: 2px;"
                >
                  mdi-cowboy
                </v-icon>
                <v-icon v-else-if="client.isAdmin" 
                   x-small style="opacity: 70%; margin-top: 2px;"
                >
                  mdi-account-tie
                </v-icon>
                <v-icon v-else 
                  x-small style="opacity: 70%; margin-top: 2px;"
                >
                  mdi-account
                </v-icon>
                <!-- TODO: just write a v-if v-else statement FFS -->
                <p class="mb-0 ml-1 text-truncate" :style="`margin-top: 2px; max-width: ${client.sessionID === sessionID ? '86px' : '130px'}`">
                  {{ client.firstName  + " " + client.lastName }} 
                </p>
              </div>
            </div>

            <!-- MY UI -->
            <template v-if="client.sessionID === sessionID">
              <div>
                <template v-if="connectionStatus === 'CONNECTED'">
                  <portal to="call-button">
                    <v-btn @click.prevent.stop="$store.commit('SET_CAN_HEAR_AUDIO', false); leaveConferenceRoom()"
                      small dark fab color="white"
                    >
                      <v-icon color="green">mdi-volume-high</v-icon>
                    </v-btn>
                  </portal>
                </template>

                <template v-else-if="connectionStatus === 'ERROR'">
                  <p class="red--text">ERROR</p> 
                  <!-- TODO: show troubleshoot popup -->
                </template>
              </div>
              <!-- End of video call buttons -->

              <v-spacer/> 

              <!-- Display a call icon if not connected -->
              <template v-if="connectionStatus !== 'CONNECTED'">
                <portal to="call-button">
                  <v-btn @click="joinConferenceRoom()" 
                    :loading="connectionStatus === 'CONNECTING'"
                    color="rgb(62, 66, 66)" fab dark class="white--text" small
                  >
                    <v-icon>
                      {{ connectionStatus === 'CONNECTED' ? 'mdi-volume-high' : 'mdi-volume-off' }}
                    </v-icon>
                  </v-btn>
                </portal>
              </template>
              
              <!-- participants && participants.local -->
              <template v-if="connectionStatus === 'CONNECTED'">
                <v-btn @click.stop.prevent="toggleMic(); $store.commit('SET_IS_MIC_ON', !isMicOn)" 
                  small fab :color="`${ isMicOn ? 'white' : 'white' }`"
                  active-class=""
                >
                  <v-icon v-if="isMicOn" color="grey darken-3">
                    mdi-microphone
                  </v-icon>
                  <v-icon v-else color="red">
                    mdi-microphone-off
                  </v-icon>
                </v-btn>
              </template>

              <!-- Switch board dropdown -->
              <portal-target name="right-side-of-my-participant-name" style="margin-left: 5px;">

              </portal-target>

              <!-- 
                <video/> will be injected here VIA `div.appendChild()
              -->

            </template>
            <!-- END OF MY OWN UI -->

            <!-- OTHER PARTICIPANT UI -->
            <template v-else>
              <v-spacer/>

              <v-icon v-if="participants.hasOwnProperty( firestoreIDToDailyID[client.sessionID] )" 
                small :color="`${participants[ firestoreIDToDailyID[client.sessionID] ].audio ? 'grey darken-3' : 'red'}`">
                {{ participants[ firestoreIDToDailyID[client.sessionID] ].audio ? 'mdi-microphone' : 'mdi-microphone-off' }}
              </v-icon>
              <!-- If I'm not connected, but someone else is connected, I'll see that their connected in a phone/voice chat-->
              <v-icon v-else-if="client.canHearAudio" color="green" small>
                mdi-volume-high
              </v-icon>
              <v-icon v-else-if="!client.canHearAudio && connectionStatus === 'CONNECTED'" small color="red darken-2">
                mdi-volume-off
              </v-icon>
              <!-- If I'm not in voice chat, I don't really particularly care about other people are also not connected -->

              <div
                @click="scrollToThisBoard(client.currentBoardID)"
                :style="`color: ${client.currentPenColor ? client.currentPenColor : 'white' } !important; 
                         width: 42px; 
                         height: 32.5px; 
                         display: flex; 
                         justify-content: center; 
                         align-items: center; 
                         background-color: rgb(62, 66, 66); 
                         margin-left: 5px; 
                         margin-right: 12px;
                         box-shadow: 0 10px 20px rgba(0,0,0,0.19), 0 6px 6px rgba(0,0,0,0.23);
                       `"
              >
                {{ client.currentBoardNumber }}
              </div>
            </template>
          </div>
        </div>
        <!-- </div> -->
      </template>
    </portal>
  </div>
</template>

<script>
/** 
 * KNOWN ISSUES: 
 *   - Status breakpoints: error
 *   - Sometimes the laptop just keeps loading infinitely
 *   - Inconsistent video constraints / aspect ratios: investigate user media constraints
 * 
 * @see API guide https://docs.daily.co/reference#%EF%B8%8F-createcallobject
 * @see Github demo code https://github.com/daily-demos/call-object-react/blob/main/src/components/Call/Call.js
 */

/**
  To get realtime support, visit https://www.daily.co/contact/support
  Helpful gist: https://gist.github.com/kwindla/9fd662a83e190e6dd003869282ff0d99

  Daily's CEO also replies on StackOverflow : ) 
 **/
import { API_KEY_SECRET } from "@/dailyCreds.js";
import { mapState } from "vuex"; 
import MicStreamsInitializer from '@/mixins/MicStreamsInitializer.js'
import VideoTroubleshootPopup from '@/components/VideoTroubleshootPopup.vue'
import VideoConferenceRoomGeneralErrorPopup from '@/components/VideoConferenceRoomGeneralErrorPopup.vue'
import DailyIframe from '@daily-co/daily-js';

const PARTICIPANT_EVENTS = ["participant-joined", "participant-updated", "participant-left"]; 

export default {
  props: {
    roomID: {
      type: String,
      required: true
    },
    roomIDToParticipants: {
      type: Object,
      required: true
    }
  },
  mixins: [
    MicStreamsInitializer
  ],
  components: {
    VideoTroubleshootPopup,
    VideoConferenceRoomGeneralErrorPopup
  },
  data () {
    return {
      isDestroyed: false,
      isSharingScreen: false,
      connectionFailureDetector: null,
      isShowingVideoTroubleshootPopup: false,
      isShowingGeneralErrorPopup: false,
      feedbackForUser: ''
    };
  },
  computed: {
    ...mapState([
      "user",
      "canHearAudio", // TODO: rename it to "willAutoJoin" or something
      "CallObject",
      "connectionStatus",
      "participants",
      "roomIDtoParticipants",
      "firestoreIDToDailyID",
      "activeSpeakerDailyID",
      'micStream'
    ]),
    allClients () { 
      if (!this.roomIDToParticipants) return; 
      else return this.roomIDToParticipants[this.roomID]; 
    },
    numOfPeopleInRoom () {
      if (!this.allClients) return 0
      else return this.allClients.length
    },
    sessionID () { return this.$store.state.session.currentID; },
    isMicOn () { return this.participants.local.audio; },
    isCamOn () { return this.participants.local.video; }
  },
  watch:  {
    // after initially joining a room, Analia will auto-join subsequent rooms 
    connectionStatus: {
      immediate: true,
      handler (newValue) {
        if (newValue === 'DISCONNECTED' && this.canHearAudio) {
          this.joinConferenceRoom(); 
        }
      }
    },
    numOfPeopleInRoom (newVal, oldVal) {
      if (newVal > oldVal) {
        const joinSound = new Audio(require('@/assets/state-change_confirm-up.wav'))
        joinSound.play()
      } else {
        const leaveSound = new Audio(require('@/assets/state-change_confirm-down.wav'))
        leaveSound.play()
      }
    }
  },
  created () {
    // console.log('video room created', this.roomID)
  },
  async destroyed () {
    // console.log('videoConferenceRoom destroyed, room =', this.roomID)
    this.isDestroyed = true; 
    // case 1 (sequential): the user leaves after connection has been resolved
    if (this.connectionStatus === "CONNECTED") {
      this.cleanUpCallObject(); 
    }
    // otherwise, as soon as the user connects to the room via `joinConferenceRoom`, 
    // the `isDestroyed` flag will initialize the cleanup straight away.
  },
  methods: {
    // That's all we need to do, 
    // because the intersection API above will call `updateCurrentBoardID`, and `currentBoardID` => `currentBoardNumber`
    scrollToThisBoard (boardID) {
      const blackboardElement = document.getElementById(boardID)
      blackboardElement.scrollIntoView({ behavior: 'smooth', block: 'start' })
    },
     /**
     * @see https://www.daily.co/blog/video-call-api-tutorial-the-rooms-family-of-endpoints/
     * @see https://docs.daily.co/reference#room-configuration
     */
    async createConferenceRoom () {
      return new Promise(async (resolve) => {
        try {
          // const SECONDS_IN_TWO_HOURS = 2 * 60 * 60;
          // for some reason, SECONDS_IN_TWO_HOURS gives more than two hours...lasts at least 6 hours, so divide it by 24 to make it half an hour
          const SECONDS_IN_HALF_HOUR = 0.5 * 60 * 60; 

          const response = await fetch("https://api.daily.co/v1/rooms", {
            "method": "POST",
            "headers": {
              "content-type": "application/json", // remove this allows your room to beb created
              "Authorization": "Bearer " + API_KEY_SECRET
            },
            // mode: "cors",
            body: JSON.stringify({
              name: this.roomID,
              properties: {
                exp: Math.round(Date.now() / 1000) + SECONDS_IN_HALF_HOUR,
                eject_at_room_exp: true,
                start_video_off: true
              }
            })
          }); 
          const room = await response.json(); 
          if (room.error === "invalid-request-error" && room.info === `a room named ${this.roomID} already exists`) {
            resolve({ url: `https://feynman.daily.co/${this.roomID}` }); 
          } else {
            console.log('room =', room)
            // alert('Cannot create conference room')
            resolve(room); 
          }
        } catch (error) {
          // TODO: handle error explicitly
          console.error(error);
          alert(error);
          resolve({ url: '' });
        } 
      });
    },
    async joinConferenceRoom () {
      const FIFTEEN_SECONDS_IN_MILLISECONDS = 15 * 1000; 
      this.connectionFailureDetector = setTimeout(() => {
        this.feedbackForUser = 'Connection timed out after 15 seconds'
        this.isShowingGeneralErrorPopup = true
        console.error("CAN'T CONNECT DESPITE 15 seconds"); 
      }, FIFTEEN_SECONDS_IN_MILLISECONDS);

      // quickfix
      if (this.$store.state.isMusicPlaying) {
        const { musicAudioElement } = this.$store.state; 
        if (musicAudioElement) musicAudioElement.pause(); 
        this.$store.commit("SET_IS_MUSIC_PLAYING", false);
      }
      this.$store.commit("SET_CONNECTION_STATUS", "CONNECTING");
      try {
        const conferenceRoom = await this.createConferenceRoom(); 

        if (!this.CallObject) {
          await this.$_initializeMicStreams()
          const [ micMediaStreamTrack ] = this.micStream.getAudioTracks()
          this.$store.commit(
            'SET_CALL_OBJECT', 
            DailyIframe.createCallObject({
              audioSource: micMediaStreamTrack, // how to destroy it // enable and disable it as you connect / disconnect from phone calls,
              videoSource: micMediaStreamTrack // clog up the video source so it doesn't activate the user's camera lights 
            }) 
          )
          this.initializeCallObject()
        }
        await this.CallObject.join({
          url: conferenceRoom.url, // `https://feynman.daily.co/${'Ly77BmJeKWudV1FJISnD'}`,
          userName: this.sessionID
        }); 
        if (this.isDestroyed) { 
          // case 2 (non-sequential): the user joined this room but left before 
          // the join request even finished; now that it has finished, we clean up the room immediately
          this.leaveConferenceRoom();
        } else {
          this.$store.commit("SET_CONNECTION_STATUS", "CONNECTED");
          this.$store.commit("SET_CAN_HEAR_AUDIO", true);
        }
      } catch (error) {
        this.$store.commit('SET_CONNECTION_STATUS', 'ERROR')
        this.feedbackForUser = error
        this.isShowingGeneralErrorPopup = true
        console.error(error);
      } finally {
        clearTimeout(this.connectionFailureDetector); 
      }
    },
    toggleMic () {
      this.CallObject.setLocalAudio(! this.isMicOn);
    },
    toggleCam () {
      this.CallObject.setLocalVideo(! this.isCamOn); 
    },
    toggleScreen () {
      if (! this.isSharingScreen) this.CallObject.startScreenShare(); 
      else this.CallObject.stopScreenShare(); 
      this.isSharingScreen = ! this.isSharingScreen; 
    },
    async leaveConferenceRoom () {
      const camVideoElems = document.querySelectorAll("video");
      for (const camVideoElem of camVideoElems) {
        camVideoElem.remove(); 
      }
      const micAudioElems = document.querySelectorAll("audio"); 
      for (const micAudioElem of micAudioElems) {
        micAudioElem.remove(); 
      }
      this.cleanUpCallObject();
      // left room because the user hung-up rather than just moving between rooms
      if (!this.canHearAudio) {
        for (const track of this.micStream.getAudioTracks()) {
          track.stop()
          this.$store.commit('SET_MIC_STREAM', null)
          this.$store.commit('SET_CALL_OBJECT', null)
        }
      }
    },
    async cleanUpCallObject () {
      this.$store.commit("SET_CONNECTION_STATUS", "DISCONNECTING"); 
      await this.CallObject.leave(); 
      this.$store.commit("SET_PARTICIPANTS", {}); 
      this.$store.commit("SET_CONNECTION_STATUS", "DISCONNECTED");
      this.$store.commit("SET_ACTIVE_SPEAKER_DAILY_ID", "");
    },

    // NEW CODE
    initializeCallObject () {
      // initialize event listeners (documentation: https://docs.daily.co/reference#events)
      const ONE_HUNDRED_MILLISECONDS = 100; 
      for (const event of PARTICIPANT_EVENTS) {
        this.CallObject.on(
          event, 
          _.throttle(this.maintainParticipantsCorrectness, ONE_HUNDRED_MILLISECONDS) 
        ); 
      }
      this.CallObject.on("track-started", this.mountNewTrack);
      this.CallObject.on("track-stopped", this.unmountTrack);
      this.CallObject.on("active-speaker-change", ({ activeSpeaker }) => {
        this.$store.commit("SET_ACTIVE_SPEAKER_DAILY_ID", activeSpeaker.peerId);
      });

      // handle specifically because it's common for users to use Zoom 
      this.CallObject.on("camera-error", (payload) => {
        this.isShowingVideoTroubleshootPopup = true; 
        console.error(payload); 
        console.log("CallObject state =", this.CallObject.meetingState()); 
      });

      // general errors
      this.CallObject.on("load-attempt-failed", ({ action, errorMsg }) => {
        this.isShowingGeneralErrorPopup = true; 
        this.feedbackForUser = action + ": " + errorMsg; 
      }); 
      this.CallObject.on("error", ({ action, errorMsg }) => {
        this.isShowingGeneralErrorPopup = true; 
        this.feedbackForUser = action + ": " + errorMsg; 
      });
    },
    maintainParticipantsCorrectness () {
      this.$store.commit("SET_PARTICIPANTS", { ...this.CallObject.participants() }); 
      this.$store.commit("SET_FIRESTORE_ID_TO_DAILY_ID", {});
      const temp = {}; 
      for (const participant of Object.values(this.participants)) {
        const { user_name, user_id } = participant; 
        // check if it Firestore participant.sessionID is binded to the Daily user_name
        if (user_name) {
          temp[user_name] = user_id; 
        }
      }
      this.$store.commit("SET_FIRESTORE_ID_TO_DAILY_ID", temp)
    },
    async mountNewTrack ({ track, participant }) {
      switch (track.kind) {
        case "video": 
          const v = document.createElement("video"); 
          v.srcObject = new MediaStream([track]);
          v.setAttribute("id", track.id);
          v.setAttribute("muted", true); 
          v.setAttribute("autoplay", true); 
          v.setAttribute("playsinline", true); // without it, iOS forces video to play in fullscreen
          v.style.width = "100%"; 
          v.setAttribute("z-index", 2); // not great
          
          if (this.isScreenTrack(track, participant)) {
            document.getElementById("screenshare-container").appendChild(v); 
          } 
          else if (! participant.user_name) {  
            // handles edge case that when joining initially, user_name is not populated by Daily API
            // user_name maps to sessionID
            v.classList.add("mirror-flip");
            document.getElementById("my-local-video").appendChild(v); 
          } 
          else if (participant.user_name === this.sessionID) { // TODO: refactor. Handles the case where I re-share my camera, but now my user_name is defined
            v.classList.add("mirror-flip");
            document.getElementById("my-local-video").appendChild(v); 
          } 
          else { 
            document.getElementById(participant.user_name).appendChild(v);
          }
          break; 

        case "audio": 
          if (participant.local) return; 
          else {
            const audioElement = document.createElement("audio"); 
            audioElement.srcObject = new MediaStream([track]); 
            audioElement.setAttribute("id", "audio" + participant.user_id); 
            audioElement.setAttribute("playsinline", true); 
            audioElement.setAttribute("autoplay", true); 

            document.getElementById("container-for-audio-elements").appendChild(audioElement);
          }
          break;
      }
    },
    async unmountTrack ({ track, participant }) {
      const trackElement = document.getElementById(track.id); 
      if (trackElement) { // sometimes the trackElement unexpectedly doesn't exist, though the error is harmless
        trackElement.srcObject = null; 
        trackElement.remove(); 
      }
    },
    isScreenTrack (track, participant) {
      if (participant) if (participant.screen && participant.screenVideoTrack.id === track.id) return true; 
      // screen:0:0
      // window:263938:1
      // web-contents-media-stream://556:4
      return ["screen", "window"].includes(track.label.substring(0, 6));
    }
  }
};
</script>

<style>
.video-overlay {
  /* position: absolute; */
  /* top: 116px; */
  /* bottom: 0px; */
  background-color: white;
  opacity: 0.9;
  width: 100%;
  z-index: 5;
}
</style>