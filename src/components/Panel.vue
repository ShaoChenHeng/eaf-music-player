<template>
  <div
    class="panel"
    :style="{ 'background': backgroundColor }">
    <div class="coverBox">
      <img
        v-if="currentCover"
        class="cover"
        :src="currentCover"/>
    </div>
    <div
      class="info"
      :style="{ 'color': foregroundColor }">
      <div>
        {{ name }}
      </div>
      <div>
        {{ artist }}
      </div>
    </div>
    <div
      class="control"
      :style="{ 'color': foregroundColor }">
      <img
        class="repeat"
        :style="{ 'color': foregroundColor }"
        :src="fileIconPath(playOrderIcon)"
        @click="togglePlayOrder"
      />
      <img
        class="backward"
        :style="{ 'color': foregroundColor }"
        :src="fileIconPath(stepBackwardIcon)"
        @click="playPrev"
      />
      <img
        class="play"
        :style="{ 'color': foregroundColor }"
        :src="fileIconPath(playIcon)"
        @click="togglePlayStatus"
      />
      <img
        class="forward"
        :style="{ 'color': foregroundColor }"
        :src="fileIconPath(stepForwardIcon)"
        @click="playNext"
      />
      <div class="current-time">
        {{ currentTime }}
      </div>
      /
      <div class="duration">
        {{ duration }}
      </div>
    </div>
    <div class="visual">
      <audio ref="player">
        <source :src="currentTrack">
      </audio>
      <av-bars
        class="visual-bar"
        ref-link="player"
        caps-color="#FFF"
        :bar-color="getBarColors()"
        :caps-height="2"
      />
    </div>
  </div>
</template>

<script>
 import { mapState } from "vuex";
 import albumArt from "album-art"

 export default {
   name: 'Panel',
   data() {
     return {
       currentTime: "",
       currentCover: "",
       iconCacheDir: "",
       pathSep: "",
       duration: "",
       name: "",
       artist: "",
       backgroundColor: "",
       foregroundColor: "",
       /* Download icon from https://www.iconfont.cn/collections/detail?spm=a313x.7781069.0.da5a778a4&cid=18739 */
       stepBackwardIcon: "step-backward",
       stepForwardIcon: "step-forward",
       playIcon: "play-circle",
       playOrderIcon: "list",
       iconKey: 1,
       port: ""
     }
   },
   computed: mapState([
     "currentTrack",
     "currentTrackIndex",
     "fileInfos"
   ]),
   watch: {
     "fileInfos": function() {
       if (this.playOrderIcon === "random") {
         this.playRandom();
       } else {
         if (this.$refs.player.paused) {
           this.playItem(this.fileInfos[this.currentTrackIndex]);
         }
       }
     },
     
     currentTime: function(newVal) {
       this.$emit('getCurrentTime', newVal);
     }
   },
   props: {
   },
   mounted() {
     window.initPanel = this.initPanel;
     window.initPort = this.initPort;
     window.forward = this.forward;
     window.backward = this.backward;
     window.togglePlayStatus = this.togglePlayStatus;
     window.playNext = this.playNext;
     window.playPrev = this.playPrev;
     window.playRandom = this.playRandom;
     window.togglePlayOrder = this.togglePlayOrder;
     

     let that = this;

     this.$root.$on("playItem", this.playItem);

     this.$root.$on("updatePanelInfo", this.updatePanelInfo);

     this.$refs.player.addEventListener("ended", this.handlePlayFinish);

     this.$refs.player.addEventListener('timeupdate', () => {
       that.currentTime = that.formatTime(that.$refs.player.currentTime);
       that.duration = that.formatTime(that.$refs.player.duration);
     });
     
     setTimeout(() => {
       this.ws = new WebSocket('ws://localhost:' + this.port);
     }, 500)
     
   },
   methods: {
     playItem(item) {
       this.$store.commit("updateCurrentTrack", item.path);

       this.playIcon = "pause-circle";

       this.name = item.name;
       this.artist = item.artist;

       this.$refs.player.load();
       this.$refs.player.play();

       this.currentCover = "";
       albumArt(item.artist, {album: item.album, size: 'small'}, (error, url) => {
         console.log(error, url);

         if (error) {
           this.currentCover = "";
         } else {
           this.currentCover = url;
         }
       })

       albumArt(item.artist, {album: item.album, size: 'large'}, (error, url) => {
         console.log(error, url);
         if (error) {
           this.$store.commit("updateCover", "");
         } else {
           this.$store.commit("updateCover", url);
         }
       })
       
       // Waiting server established.
       if (this.currentTrackIndex === 0) {
         setTimeout(() => {
           this.getLyric();
         }, 1000)
       } else {
         this.getLyric();
       }
     },

     onSubmit(songInfo) {
       this.ws.send(songInfo);
     },
     
     onStop() {
       this.ws.send('stop');
     },

     getLyric() {
       let currentSong = this.fileInfos[this.currentTrackIndex];
       this.onSubmit(JSON.stringify(currentSong));
       this.ws.onmessage = (event) => {
         let rawLyric = event.data;
         let lines = rawLyric.split('\n');
         let newLyric = [];
         lines.forEach((line, index) => {
           let newLine = {};
           if (!line) {
             return ;
           }
           let pattern = /\[\S*\]/g;
           let time = line.match(pattern)[0];
           let lineLyric = line.replace(time, '');
           time = time.replace(/\[/, '');
           time = time.replace(/\]/, '');
           newLine.index = index;
           newLine.time = time;
           newLine.content = lineLyric.trim();
           if (newLine.content == '') {
             newLine.content = "~";
           }
           newLine.second = (time.split(":")[0] * 60 + parseFloat(time.split(":")[1])).toFixed(0);
           newLyric.push(newLine);
         })
         this.$store.commit("updateLyric", newLyric);
       }
     },
     
     updatePanelInfo(name, artist) {
       this.name = name;
       this.artist = artist;
     },

     togglePlayOrder() {
       if (this.playOrderIcon === "list") {
         this.playOrderIcon = "random";
       } else if (this.playOrderIcon === "random") {
         this.playOrderIcon = "repeat";
       } else if (this.playOrderIcon === "repeat") {
         this.playOrderIcon = "list";
       }
     },

     handlePlayFinish() {
       if (this.playOrderIcon === "list") {
         this.playNext();
       } else if (this.playOrderIcon === "random") {
         this.playRandom();
       } else if (this.playOrderIcon === "repeat") {
         this.playAgain();
       }
     },

     initPanel(playOrderIcon, backgroundColor, foregroundColor, iconCacheDir, pathSep) {
       this.playOrderIcon = playOrderIcon;
       this.backgroundColor = backgroundColor;
       this.foregroundColor = foregroundColor;
       this.iconCacheDir = iconCacheDir;
       this.pathSep = pathSep;

       this.iconKey = new Date();
     },

     initPort(port) {
       this.port = port;
     },
     
     fileIconPath(iconFile) {
       return this.iconCacheDir + this.pathSep + iconFile + ".svg" + "?cache=" + this.iconKey;
     },

     getBarColors() {
       return [this.backgroundColor, this.foregroundColor, this.foregroundColor]
     },

     formatTime(seconds) {
       if (isNaN(seconds)) {
         return "00:00"
       } else {
         var minutes = Math.floor(seconds / 60);
         minutes = (minutes >= 10) ? minutes : "0" + minutes;
         seconds = Math.floor(seconds % 60);
         seconds = (seconds >= 10) ? seconds : "0" + seconds;
         return minutes + ":" + seconds;
       }
     },

     forward() {
       this.$refs.player.currentTime += 10;
     },

     backward() {
       this.$refs.player.currentTime -= 10;
     },

     togglePlayStatus() {
       if (this.$refs.player.paused) {
         this.$refs.player.play();
         this.playIcon = "pause-circle";
       } else {
         this.$refs.player.pause();
         this.playIcon = "play-circle";
       }
     },

     playPrev() {
       var currentTrackIndex = this.currentTrackIndex;

       if (currentTrackIndex > 0) {
         currentTrackIndex -= 1;
       } else {
         currentTrackIndex = this.fileInfos.length - 1;
       }

       this.playItem(this.fileInfos[currentTrackIndex]);
     },

     playNext() {
       var currentTrackIndex = this.currentTrackIndex;

       if (currentTrackIndex < this.fileInfos.length - 1) {
         currentTrackIndex += 1;
       } else {
         currentTrackIndex = 0;
       }

       this.playItem(this.fileInfos[currentTrackIndex]);
     },

     playRandom() {
       var min = 0;
       var max = this.fileInfos.length;
       var randomIndex = Math.floor(Math.random() * (max - min + 1)) + min;

       this.playItem(this.fileInfos[randomIndex]);
     },

     playAgain() {
       this.playItem(this.fileInfos[this.currentTrackIndex]);
     }
   }
 }
</script>

<style scoped>
 .panel {
   position: absolute;
   bottom: 0;

   width: 100%;

   box-shadow: 0px -4px 3px rgba(30, 30, 30, 0.2);

   display: flex;
   flex-direction: row;
   align-items: center;
 }

 .coverBox {
   width: 60px;
   margin-left: 30px;
 }
 
 .cover {
   width: 100%;
 }

 .info {
   width: 30%;
   user-select: none;
   padding-left: 30px;

   overflow: hidden;
   white-space: nowrap;
   text-overflow: ellipsis;
 }

 .control {
   display: flex;
   flex-direction: row;
   align-items: center;
   width: 40%;
   justify-content: center;
 }

 .visual {
   width: 30%;
   padding-right: 30px;
 }

 .visual-bar {
   display: flex;
   justify-content: flex-end;
 }

 .backward {
   cursor: pointer;
   height: 48px;
 }

 .forward {
   cursor: pointer;
   height: 48px;
 }

 .play {
   margin-left: 10px;
   margin-right: 10px;
   cursor: pointer;
   height: 60px;
 }

 .play-order {
   margin-right: 20px;
   height: 60px;
 }

 .repeat {
   margin-right: 20px;
   height: 24px;
 }
 
 .current-time {
   margin-left: 20px;
   margin-right: 5px;
 }

 .duration {
   margin-left: 5px;
 }
</style>
