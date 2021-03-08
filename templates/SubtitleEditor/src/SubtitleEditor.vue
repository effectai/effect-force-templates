<template>
<div class="subtitle-editor">
  <div style="align-items: flex-start; display:flex;">
    <div class="subtitle-container">
      <div v-for="(sub, index) in subs" :key="sub.id">
        <div style="display:flex; justify-content: center; align-items: center" class="subtitle" :class="{'active': index == activeSubIndex, 'focus': index == focusSubIndex}">
          <div style="margin-right:10px; width: 60px;">
            <div><b class="seconds-link" @click="jumpToSeconds(sub.startTime)">{{sub.startTime}}s</b></div>
            <div><b class="seconds-link" @click="jumpToSeconds(sub.endTime)">{{sub.endTime}}s</b></div>
            <div style="margin: 5px; width: 50px;">&nbsp;<span v-show="index == focusSubIndex || index == activeSubIndex">original:</span></div>
          </div>
          <div style="width: 500px; max-width: 100%; min-height: 60px;" :class="{'sub-warning': sub.error == 1, 'sub-error': sub.error == 2}">
            <div v-if="sub.hide && false" @click="activateSub(sub.startTime, index)" class="fake-textarea">{{sub.text}}</div>
            <textarea v-else :ref="'sub-'+sub.id" :value="sub.text" @input="updateSubText(sub, $event)"  @blur="focusSubIndex = null;getSrt()" @focus="activateSub(sub.startTime, index)" style="height: 43px;"></textarea>
            <div v-if="index == focusSubIndex || index == activeSubIndex">{{sub.originalText}}</div>
          </div>
        </div>
      </div>
    </div>
    <div id="player-container" class="player-container">
      <video ref="player" id="player" playsinline controls>
        <source :src="videoUrl" type="video/mp4" />
      </video>
      <div class="subtitle-video">
        <b v-if="activeSubIndex !== null && activeSubIndex >= 0 && subs[activeSubIndex]">
          <pre>{{subs[activeSubIndex].text}}</pre>
        </b>
      </div>
      <div id="timeline">
        <div v-if="duration" ref="timeline" :style="{transform: 'translateX(' + (50+currentTime*-50) + 'px)', minWidth: (duration * 50) + 'px'}" style="border-bottom: 1px solid grey; transform: translateX(50px);">
          <draggable-subtitle v-bind:key="index" v-for="(sub, index) in subs" @dragging="(x, y) => subTiming(index, x)" @resizing="(x, y, width, height) => subTiming(index, x, width)" :handles="(!ids || ids.length == 0) ? ['ml','mr'] : (index == 0) ? ['mr'] : (index == ids.length - 1) ? ['ml'] : ['ml', 'mr']" :id="sub.id" @activated="scrollToSub(sub.id)" @resizestop="getSrt" @dragstop="getSrt" :ref="'subt-'+sub.id" :draggable="!ids || ids.length == 0 || (index != 0 && index != ids.length - 1)" :parent="true" axis="x" class-name="timeline-sub" :class="{'playing': index == activeSubIndex}" :min-width="50" :x="sub.x" :h="50" :w="sub.w">
            <div class="sub-text">{{ sub.text }}</div>
          </draggable-subtitle>
          <div class="time">
            <div v-for="n in timings" v-bind:key="n" :style="{left: (n) * 50 + 'px'}"><span>{{ n | minutes }}</span></div>
          </div>
        </div>
        <div id="timeline-progress" style="left: 50px"></div>
      </div>
    </div>
    <textarea style="visibility:hidden;position:absolute;width:0" name="subtitles" v-model="srt"></textarea>
  </div>
  <div class="comments" v-if="comments">
    <b>Validator Comment</b>
    <div style="
                background: #ffffbe;
                font-size: 18px;
                margin: 0 30px;
                border-radius: 10px;
                padding: 5px 15px;
                ">{{comments}}</div>
  </div>
  <div style="text-align:center">
    <a class="btn btn-primary btn-outlined btn-control rewind" @click="shiftSeconds(-10)">&#x3C; Rewind (-10s)</a>
    <a class="btn btn-primary btn-outlined btn-control play" @click="player.togglePlay()">Play / Pause</a>
    <a class="btn btn-primary btn-outlined btn-control forward" @click="shiftSeconds(10)">&#x3E; Forward (+10s)</a>
  </div>
</div>
</template>

<script>
import VueDraggableResizable from 'vue-draggable-resizable'
import Plyr from 'plyr';

function range(min, max) {
  var len = max - min + 1;
  var arr = new Array(len);
  for (var i = 0; i < len; i++) {
    arr[i] = min + i;
  }
  return arr;
}

const parser = (function() {
  var pItems = {};
  
  /**
   * Converts SubRip subtitles into array of objects
   * [{
   *     id:        `Number of subtitle`
   *     startTime: `Start time of subtitle`
   *     endTime:   `End time of subtitle
   *     text: `Text of subtitle`
   * }]
   *
   * @param  {String}  data SubRip suntitles string
   * @param  {Boolean} ms   Optional: use milliseconds for startTime and endTime
   * @return {Array}
   */
  pItems.fromSrt = function(data, originalSrt, hashCode) {
    data = data.replace(/\r/g, '');
    var regex = /(\d+)\n(\d{2}:\d{2}:\d{2},\d{3}) --> (\d{2}:\d{2}:\d{2},\d{3})/g;
    data = data.split(regex);
    data.shift();
    let storageSrt;
    if (hashCode && localStorage.getItem('srt' + hashCode)) {
      storageSrt = localStorage.getItem('srt' + hashCode);
      storageSrt = storageSrt.replace(/\r/g, '');
      storageSrt = storageSrt.split(regex);
      storageSrt.shift();
    }
    if (originalSrt) {
      originalSrt = originalSrt.replace(/\r/g, '');
      originalSrt = originalSrt.split(regex);
      originalSrt.shift();
    }
    
    var items = [];
    for (var i = 0; i < data.length; i += 4) {
      let item = {
        id: data[i].trim(),
        startTime: timeMs(data[i + 1].trim()) / 1000.0,
        endTime: timeMs(data[i + 2].trim()) / 1000.0,
        text: data[i + 3].trim(),
        originalText: data[i + 3].trim()
      };
      if (storageSrt) {
        if (storageSrt[i] && storageSrt[i].trim() === item.id) {
          item.text = storageSrt[i + 3].trim()
        }
      }
      if (originalSrt) {
        if (originalSrt[i] && originalSrt[i].trim() === item.id) {
          item.originalText = originalSrt[i + 3].trim()
        }
      }
      items.push(item);
    }
    
    return items;
  };
  
  /**
   * Converts Array of objects created by this module to SubRip subtitles
   * @param  {Array}  data
   * @return {String}      SubRip subtitles string
   */
  pItems.toSrt = function(data) {
    if (!(data instanceof Array)) return '';
    var res = '';
    
    for (var i = 0; i < data.length; i++) {
      var s = data[i];
      
      res += s.id + '\r\n';
      res += msTime(parseInt(s.startTime * 1000, 10)) + ' --> ' + msTime(parseInt(s.endTime * 1000, 10)) + '\r\n';
      res += s.text.replace('\n', '\r\n') + '\r\n\r\n';
    }
    
    return res;
  };
  
  var timeMs = function(val) {
    var regex = /(\d+):(\d{2}):(\d{2}),(\d{3})/;
    var parts = regex.exec(val);
    
    if (parts === null) {
      return 0;
    }
    
    for (var i = 1; i < 5; i++) {
      parts[i] = parseInt(parts[i], 10);
      if (isNaN(parts[i])) parts[i] = 0;
    }
    
    // hours + minutes + seconds + ms
    return parts[1] * 3600000 + parts[2] * 60000 + parts[3] * 1000 + parts[4];
  };
  
  var msTime = function(val) {
    var measures = [3600000, 60000, 1000];
    var time = [];
    
    for (var i in measures) {
      var res = (val / measures[i] >> 0).toString();
      
      if (res.length < 2) res = '0' + res;
      val %= measures[i];
      time.push(res);
    }
    
    var ms = val.toString();
    if (ms.length < 3) {
      for (i = 0; i <= 3 - ms.length; i++) ms = '0' + ms;
    }
    
    return time.join(':') + ',' + ms;
  };
  
  return pItems;
})();

export default {
  el: '#task',
  props: ['srtUrl', 'videoUrl', 'ids'],
  data() {
    return {
      comments: null,
      parser: parser,
      hashCode: null,
      loop: true,
      currentTime: 0,
      duration: null,
      srt: null,
      timings: null,
      player: null,
      subs: null,
      activeSubIndex: null,
      focusSubIndex: null
    }
  },
  components: {
    'draggable-subtitle': VueDraggableResizable
  },
  computed: {
    
  },
  methods: {
    updateSubText: function(sub, event) {
      let oldText = sub.text;
      let newText = event.target.value;
      let oldLines = oldText.split("\n");
      let lines = newText.split("\n");
      
      if(lines.length > 2) {
        if(newText.startsWith("\n") || newText.endsWith("\n")) {
          newText = newText.trim();
        } else if(oldLines.length > 1) {
          if(newText.includes(oldLines[0] + "\n")) {
            newText = newText.replace(oldLines[0]+"\n", oldLines[0]);
          } else if(newText.includes("\n" + oldLines[1])) {
            newText = newText.replace("\n"+oldLines[1], oldLines[1]);
          } else {
            newText = newText.replace("\n","");
          }
        } else {
          newText = newText.replace("\n","");
        }
      }
      let newLines = newText.split("\n");
      let error = 0;
      for (let line of newLines) {
        if (line.length > 40 || newLines.length > 2) {
          error = 2;
          break;
        } else if(line.length > 30) {
          error = 1;
        }
      }
      sub.text = newText;  // Actual assignment
      this.$set(sub, 'error', error);
    },
    getSrt: function() {
      let srt = this.subs;
      if (!srt) return null;
      const parsed = parser.toSrt(srt);
      
      localStorage.setItem('srt' + this.hashCode, parsed);
      this.srt = parsed;
    },
    dragTimeline: function(elmnt) {
      var pos1 = 0,
          pos2 = 0,
          jumpPos = 0,
          dragged = false;
      const dragMouseDown = (e) => {
        if (e.target !== e.currentTarget) {
          return;
        }
        e = e || window.event;
        e.preventDefault();
        // get the mouse cursor position at startup:
        pos2 = e.clientX;
        document.onmouseup = closeDragElement;
        // call a function whenever the cursor moves:
        this.player.pause();
        document.onmousemove = elementDrag;
      }
      
      const elementDrag = (e) => {
        e = e || window.event;
        e.preventDefault();
        // calculate the new cursor position:
        pos1 = pos2 - e.clientX;
        pos2 = e.clientX;
        // set the element's new position:
        const transformStyle = elmnt.style.transform
        let translateX = parseFloat(transformStyle.replace(/[^-\d.]/g, '')) - parseFloat(pos1);
        
        jumpPos = ((translateX) - 50) / -50.0;
        dragged = true;
        if (translateX > 50) translateX = 50;
        if (translateX < (this.duration * -50)) translateX = (this.duration * -50);
        translateX = "translateX(" + translateX + "px)";
        elmnt.style.transform = translateX;
      }
      
      const closeDragElement = () => {
        // stop moving when mouse button is released:
        if (dragged) {
          this.jumpToSeconds(jumpPos);
          dragged = false;
        }
        document.onmouseup = null;
        document.onmousemove = null;
      }
      elmnt.onmousedown = dragMouseDown;
      
      
    },
    subTiming: function(index, x, width) {
      this.player.pause();
      this.$set(this.subs[index], 'x', parseInt(x));
      const startTime = parseFloat((this.subs[index].x / 50.0).toFixed(3));
      if (width) {
        this.$set(this.subs[index], 'w', parseInt(width));
      }
      const endTime = parseFloat(((this.subs[index].x + this.subs[index].w) / 50.0).toFixed(3));
      this.$set(this.subs[index], 'startTime', startTime);
      this.$set(this.subs[index], 'endTime', endTime);
      
      
      
      if (this.subs[index - 1] && this.subs[index].x < (this.subs[index - 1].x + this.subs[index - 1].w)) {
        if (this.subs[index].x - this.subs[index - 1].x >= 50) {
          this.$set(this.subs[index - 1], 'w', parseInt(this.subs[index].x - this.subs[index - 1].x));
          this.$set(this.subs[index - 1], 'endTime', startTime);
        }
      }
      
      if (this.subs[index + 1] && (this.subs[index].x + this.subs[index].w) > this.subs[index + 1].x) {
        if ((this.subs[index + 1].x + this.subs[index + 1].w) - (this.subs[index].x + this.subs[index].w) >= 50) {
          const w = parseInt(this.subs[index + 1].w - ((this.subs[index].x + this.subs[index].w) - this.subs[index + 1].x));
          this.$set(this.subs[index + 1], 'x', parseInt(this.subs[index].x + this.subs[index].w));
          this.$set(this.subs[index + 1], 'w', w);
          
          this.$set(this.subs[index + 1], 'startTime', endTime);
        }
      }
    },
    fetchSrt() {
      var request = new XMLHttpRequest();
      request.open('GET', this.srtUrl, true);
      request.send(null);
      request.onreadystatechange = () => {
        if (request.readyState === 4 && request.status === 200) {
          var request2 = new XMLHttpRequest();
          request2.open('GET', this.srtUrl, true);
          request2.send(null);
          request2.onreadystatechange = () => {
            if (request2.readyState === 4 && request2.status === 200) {
              let srt = request.responseText;
              let originalSrt = request2.responseText;
              let prevSub;
              let comments;
              try {
                // prevSub = this.previousSubmission;
                // prevSub;
              } catch (e) {
                console.error(e)
              }
              if (prevSub && prevSub.results && prevSub.results.subtitles) {
                srt = prevSub.results.subtitles
              }
              if (prevSub && prevSub.comments) {
                comments = prevSub.comments
              }
              const hashCode = s => s.split('').reduce((a, b) => (((a << 5) - a) + b.charCodeAt(0)) | 0, 0)
              let hash = hashCode(srt);
              let subs = this.parser.fromSrt(srt, originalSrt, hash);
              subs = subs.filter((sub) => {
                return !this.ids || this.ids.length === 0 || this.ids.includes(parseInt(sub.id))
              })
              let firstSub = subs[0];
              let id;
              if (this.ids.length > 0) {
                id = Math.min(...this.ids);
              }
              for (let sub of subs) {
                sub.x = parseInt(sub.startTime * 50)
                sub.w = parseInt((sub.endTime - sub.startTime) * 50)
                sub.hide = true;
                let lines = sub.text.split("\n");
                let error = 0;
                for(let line of lines) {
                  if (line.length > 50 || lines.length > 2) {
                    error = 2;
                    break;
                  } else if(line.length > 40) {
                    error = 1;
                  }
                }
                
                sub.error = error;
                if (id && sub.id == id) {
                  firstSub = sub;
                }
              }
              
              if (!window.validateMode) {
                this.subs = subs;
              }
              
              this.currentTime = firstSub.startTime;
              this.hashCode = hash;
              this.comments = comments;
              this.getSrt();
              
              this.player = new Plyr(this.$refs.player, {
                controls: ['play-large', 'play', 'progress', 'current-time', 'mute', 'volume', 'captions', 'settings', 'fullscreen'],
                keyboard: {
                  global: true,
                  focused: false,
                },
                invertTime: false,
                hideControls: false,
                fullscreen: {
                  enabled: true,
                  fallback: false,
                  iosNative: false
                }
              });
              this.player.once('loadeddata', this.onReady);
              this.player.once('ready', this.onReady);
              this.player.on('statechange', (event) => {
                if (event.detail.code == 1) {
                  const element = document.getElementById("player-container");
                  if (element.classList.contains('youtube-overlay')) {
                    element.classList.remove("youtube-overlay");
                    setTimeout(() => {
                      this.player.play();
                    }, 10);
                  }
                }
              });
              this.player.on('timeupdate', this.onTimeUpdate);
            }
          }
        }
      }
      
    },
    onReady() {
      this.player.currentTime = this.currentTime;
      this.player.muted = false;
      if (!this.player.duration || this.duration) return;
      this.duration = this.player.duration;
      
      this.$nextTick(() => {
        this.dragTimeline(this.$refs.timeline)
        for (let i = 0; i < this.$children.length; i++) {
          const tempd = this.$children[i].calcDragLimits;
          const tempr = this.$children[i].calcResizeLimits;
          
          this.$children[i].calcDragLimits = () => {
            
            const bounds = tempd();
            if (this.$children[i - 1]) {
              bounds.minLeft = this.$children[i - 1].left + 50;
              bounds.maxRight = this.$children[i].parentWidth - (bounds.minLeft + this.$children[i].width);
            }
            if (this.$children[i + 1]) {
              
              bounds.minRight = this.$children[i + 1].right + 50;
              bounds.maxLeft = this.$children[i].parentWidth - (bounds.minRight + this.$children[i].width);
            }
            return bounds
          }
          this.$children[i].calcResizeLimits = () => {
            const bounds = tempr();
            if (this.$children[i - 1]) {
              bounds.minLeft = this.$children[i - 1].left + 50;
            }
            if (this.$children[i + 1]) {
              bounds.minRight = this.$children[i].parentWidth - (this.$children[i + 1].left + this.$children[i + 1].width - 50);
            }
            return bounds
          }
        }
      });
    },
    onTimeUpdate(event, sec) {
      
      if (sec) {
        this.currentTime = sec
      } else {
        this.currentTime = this.player.currentTime;
      }
      let startTiming = Math.ceil(this.currentTime - 15);
      let endTiming = Math.ceil(this.currentTime + 15);
      if (startTiming < 0) startTiming = 0;
      if (endTiming > this.duration) endTiming = Math.ceil(this.duration + 1);
      this.timings = range(startTiming, endTiming);
      let noActiveSub = true;
      for (let i = 0; i < this.subs.length; i++) {
        if (this.currentTime < this.subs[i].endTime && this.currentTime >= this.subs[i].startTime) {
          this.activeSubIndex = i;
          noActiveSub = false;
        }
        if (this.subs[i].startTime < endTiming && this.subs[i].startTime > startTiming ||
            this.subs[i].endTime < endTiming && this.subs[i].endTime > startTiming) {
          if (this.subs[i].hide) {
            this.subs[i].hide = false;
            this.$nextTick(() => {
              const curId = this.subs[i].id;
              const curSub = this.$children[this.$children.findIndex(item => item.$attrs.id === this.subs[i].id)];
              if (curSub) {
                const tempd = curSub.calcDragLimits;
                const tempr = curSub.calcResizeLimits;
                
                curSub.calcDragLimits = () => {
                  
                  const bounds = tempd();
                  const prevSub = this.$children[this.$children.findIndex(item => item.$attrs.id == parseInt(curId) - 1)];
                  const nextSub = this.$children[this.$children.findIndex(item => item.$attrs.id == parseInt(curId) + 1)];
                  
                  if (prevSub) {
                    bounds.minLeft = prevSub.left + 50;
                    bounds.maxRight = curSub.parentWidth - (bounds.minLeft + curSub.width);
                  }
                  if (nextSub) {
                    
                    bounds.minRight = nextSub.right + 50;
                    bounds.maxLeft = curSub.parentWidth - (bounds.minRight + curSub.width);
                  }
                  return bounds
                }
                curSub.calcResizeLimits = () => {
                  const bounds = tempr();
                  const prevSub = this.$children[this.$children.findIndex(item => item.$attrs.id == parseInt(curId) - 1)];
                  const nextSub = this.$children[this.$children.findIndex(item => item.$attrs.id == parseInt(curId) + 1)];
                  if (prevSub) {
                    bounds.minLeft = prevSub.left + 50;
                  }
                  if (nextSub) {
                    bounds.minRight = curSub.parentWidth - (nextSub.left + nextSub.width - 50);
                  }
                  return bounds
                }
              }
            });
          }
          
        } else {
          this.subs[i].hide = true;
        }
      }
      if (noActiveSub) {
        this.activeSubIndex = null;
      }
    },
    jumpToSeconds(sec) {
      if (sec < 0) sec = 0;
      this.player.currentTime = sec;
      this.onTimeUpdate(null, sec);
      
      this.player.muted = false;
    },
    shiftSeconds(sec) {
      sec = this.player.currentTime + sec;
      if (sec < 0) sec = 0;
      this.player.currentTime = sec;
      this.onTimeUpdate(null, sec);
      
      this.player.muted = false;
    },
    activateSub(sec, index) {
      this.jumpToSeconds(sec);
      this.activeSubIndex = index;
      this.focusSubIndex = index;
    },
    scrollToSub(id) {
      this.$refs['sub-' + id][0].scrollIntoView({
        behavior: 'smooth'
      });
      
    },
  },
  filters: {
    minutes: function(sec) {
      let seconds = sec % 60
      let minutes = parseInt(sec / 60, 10) % 60
      if (seconds < 10) seconds = "0" + seconds;
      return minutes + ":" + seconds;
    }
  },
  mounted() {
    this.fetchSrt();
    
    this._keyListener = function(e) {
      if (e.key === "j" && (e.altKey)) {
        e.preventDefault(); // present "Save Page" from getting triggered.
        this.shiftSeconds(-10);
      }
      if (e.key === "l" && (e.altKey)) {
        e.preventDefault(); // present "Save Page" from getting triggered.
        this.shiftSeconds(10);
      }
      if (e.key === "k" && (e.altKey)) {
        e.preventDefault(); // present "Save Page" from getting triggered.
        this.player.togglePlay();
      }
    };
    
    document.addEventListener('keydown', this._keyListener.bind(this));
  },
  beforeDestroy() {
    document.removeEventListener('keydown', this._keyListener);
  }
}
</script>
<style>
#task {
    min-height: 400px;
}

.youtube-overlay iframe {
    z-index: 10;
}

#player { height: auto; }

.youtube-overlay .plyr__control,
.youtube-overlay .plyr__controls,
.youtube-overlay .plyr__poster {
    display: none !important;
}

.player-container {
    max-width: 570px;
    width: 100%;
    position: relative;
    max-height:480px;
}

.plyr {
    margin: 0 auto;
    max-width: 570px;
    max-height: 350px;
}

.subtitle-container {
    max-height: 395px;
    overflow-y: scroll;
}

.subtitle {
    margin-right: 10px;
    border-left: 2px solid transparent;
    border-bottom: 1px solid #eaeaea;
    padding-top: 10px;
    padding-right: 10px;
}

.subtitle textarea {
    font-size: 14px;
    font-family: "Roboto", sans-serif;
}

.subtitle.active {
    border-left-color: #cc181e;
}

.subtitle.focus {
    background: #def;
}

#timeline {
    display: flex;
    overflow: hidden;
    position: relative;
    padding-top: 5px;
    height: 75px;
    cursor: move;
}

.subtitle-video {
    position: absolute;
    bottom: 125px;
    margin: 0 auto;
    width: 100%;
    background: rgba(0, 0, 0, 0.6);
    color: white;
    z-index: 2;
}

.subtitle-video pre {
    margin: 0;
    font-size: 18px;
    font-family: "Roboto", sans-serif;
    white-space: pre-wrap;
    padding: 5px 10px;
}

.timeline-sub {
    border-radius: 4px;
    border: 1px solid black;
    background: #2d2d2d;
    color: white;
    font-size: 12px;
    font-family: Arial;
    white-space: pre-wrap;
    padding: 5px 5px 1px;
    box-sizing: border-box;
    overflow: hidden;
    top: 7px;
    cursor: pointer;
    position: absolute;
}

.timeline-sub .sub-text {
    overflow: hidden;
    max-height: 100%;
    pointer-events: none;
}

.handle {
    z-index: 2;
    width: 10px;
    top: 0;
    height: 100%;
    opacity: 0.3;
    background: #0b9aff;
    position: absolute;
    box-sizing: border-box;
}

.handle-ml {
    cursor: w-resize;
    left: 0;
}

.handle-mr {
    cursor: e-resize;
    right: 0;
}

.fake-textarea {
    height: 43px;
    border-radius: 2px;
    width: 100%;
    color: #164B62;
    background: #fff;
    box-sizing: border-box;
    font-size: 14px;
    border: 1px solid #D7D7D7;
    resize: none;
    font-weight: 400;
    text-align: left;
    max-width: 100%;
    margin: 0;
    padding: 5px 10px;
    cursor: text;
    white-space: pre-wrap;
    overflow-wrap: break-word;
}

.sub-warning textarea, .sub-warning .fake-textarea {
    border-color: orange !important;
    background: #ffc8a2  !important;
}
.sub-error textarea, .sub-error .fake-textarea {
    border-color: red  !important;
    background: #ff9f9f  !important;
}

#timeline-progress {
    position: absolute;
    width: 2px;
    height: 100%;
    top: 0;
    background: red;
}

.timeline-sub.playing {
    background: #b4c9de;
    color: black;
}

.timeline-sub.active {
    background: #def;
    color: black;
    z-index: 1 !important;
}

.time {
    position: absolute;
    display: flex;
    bottom: 0;
    pointer-events: none;
    font-size: 10px;
    border-bottom: 1px solid grey;
}

.time div {
    width: 50px;
    border-left: 1px solid grey;
    text-align: left;
    position: relative;
    height: 5px;
    bottom: 0px;
    position: absolute;
}

.time div:before {
    content: "";
    display: block;
    height: 3px;
    border-left: 1px solid grey;
    width: 16px;
    position: absolute;
    bottom: 0;
    left: 16px;
}

.time div:after {
    content: "";
    display: block;
    height: 3px;
    border-left: 1px solid grey;
    width: 16px;
    position: absolute;
    bottom: 0;
    left: 34px;
}

.time div span {
    position: absolute;
    left: -10px;
    top: -12px;
}

.seconds-link {
    cursor: pointer;
}

.seconds-link:hover {
    color: lightblue;
}
.btn-control {
    margin: 12px;
}
.btn-control:after {
    display:block;
    bottom:-17px;
    color: black;
    position: absolute;
    font-style: italic;
}
.btn-control.rewind:after {
    content: "alt + J";
}
.btn-control.play:after {
    content: "alt + K";
}
.btn-control.forward:after {
    content: "alt + L";
}
</style>
 
