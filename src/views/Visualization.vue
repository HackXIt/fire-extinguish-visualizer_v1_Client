<template>
  <div id="content">
    <div
      v-for="visual in visuals"
      :key="`[${visual.port.id}]-${visual.port.name}`"
      class="label"
      :style="positionLabel(visual)"
    >
      <b>Port</b>
      {{ `[${visual.port.id}]: ${visual.port.name}` }}
      <br />
      <b>Board:</b>
      {{
        `${visual.board.boardType.toUpperCase()} [${visual.IO.length} x ${
          visual.board.rpiType
        }`
      }}{{ visual.IO.length > 1 ? "s]" : "]" }}
      <br />
      <b>Name:</b>
      {{ visual.port.name }}
      <br />
      <button
        v-if="visual.board.rpiType === 'input'"
        @click="togglePolling(visual)"
      >
        {{
          `Polling: ${
            pollings[pollings.findIndex(poll => poll.id === visual.port.id)]
              .active
              ? "On"
              : "Off"
          }`
        }}
      </button>
      <br />Draggable?
      <input v-model="draggable[visual.port.id]" type="checkbox" />
    </div>
    <vue-draggable-resizable
      v-for="wrapper in flatVisuals"
      :key="wrapper.key"
      class="pin"
      :parent="true"
      :resizable="false"
      :draggable="draggable[wrapper.id]"
      :h="100"
      :w="100"
      :x="wrapper.pos * 100"
      :y="(wrapper.id - 1) * 100"
    >
      <div v-if="wrapper.ioType === 'output'">
        <button @click="sendByte(wrapper.io, wrapper.port)">
          {{
            `${wrapper.port}[${wrapper.io}] (${wrapper.state ? "Off" : "On"})`
          }}
        </button>
        <status-indicator
          :status="wrapper.state ? 'active' : 'intermediary'"
          :pulse="wrapper.state ? true : false"
        />
      </div>
      <div v-else>
        {{ `${wrapper.port}[${wrapper.io}] => ` }}
        <status-indicator
          :status="wrapper.state ? 'positive' : 'negative'"
          :pulse="wrapper.state ? true : false"
        />
        <!-- TODO Insert custom image for IO-State-->
      </div>
    </vue-draggable-resizable>
    <div class="counters" v-for="counter in counters" :key="counter.id">
      <button
        @click="switchCountdown(`vac${counter.id}`, counter.id)"
        v-text="`Toggle ${counter.name} -> ${counter.state}`"
      />
      <vac
        :ref="`vac${counter.id}`"
        :leftTime="counter.seconds * 1000"
        :autoStart="false"
      >
        <span slot="process" slot-scope="{ timeObj }">{{
          timeObj.ceil.s
        }}</span>
        <span slot="finish">Done!</span>
      </vac>
    </div>
  </div>
</template>

<script>
// import VueDraggableResizable from "vue-draggable-resizable";
import axios from "axios";
import { firePi } from "@/variables.js";
export default {
  name: "Visualization",
  data() {
    return {
      visuals: [],
      flatVisuals: {},
      draggable: {
        1: true,
        2: true,
        3: true,
        4: true
      },
      counters: [],
      paths: {
        // FIXME Network-Error when using localhost as IP
        // THIS NEEDS TO HAVE THE IP OF THE RASPBERRY PI
        // THE BROWSER REFERENCES THIS ADDRESS AND LOCALHOST POINTS TO THE CLIENT
        // Need to get some sort of static IP going or hostname
        cleanup: `http://${firePi}/cleanup`,
        shift: `http://${firePi}/shift`
      },
      pollings: [],
      response: null
    };
  },
  watch: {
    response: {
      handler(newResponse) {
        console.debug("New response received from Server");
        const visualIndex = this.visuals.findIndex(
          visual => visual.port.name === newResponse.port
        );
        const visual = this.visuals[visualIndex];
        visual.IO.forEach(io => {
          visual.pinStates[io] = newResponse.pinStates[io - 1];
          this.flatVisuals[`${visual.port.name}-${io}`].state =
            visual.pinStates[io];
          this.flatVisuals[`${visual.port.name}-${io}`].key = `${
            visual.board.boardType
          }-${io}@${visual.port.name} => ${visual.pinStates[io]}`;
        });
        console.debug(visual);
        this.visuals[visualIndex] = visual;
        this.$forceUpdate();
      },
      deep: true
    }
  },
  beforeMount() {
    console.debug("----Visualization beforeMounted----");
    if (localStorage.getItem("submissions")) {
      try {
        this.visuals = JSON.parse(localStorage.getItem("submissions"));
        this.flattenVisuals(this.visuals);
      } catch (e) {
        // NOTE This destroys localStorage data if invalid
        console.error(e);
        localStorage.removeItem("submissions");
      }
    }
    if (localStorage.getItem("delays")) {
      try {
        this.counters = JSON.parse(localStorage.getItem("delays"));
      } catch (e) {
        console.debug(e);
        localStorage.removeItem("delays");
      }
    }
    // FIXME Polling is very resource-intensive
    // A better approach would be to use WebSockets
    // That would allow for the Flask Server to send data on change
    // And the frontend would just have to wait for newResponses
    if (this.visuals) {
      this.visuals.forEach(visual => {
        if (visual.board.boardType === "om8") {
          console.debug(
            `Adding polling object for ${visual.board.boardType} @ ${
              visual.port.name
            }`
          );
          const polling = {
            id: visual.port.id,
            port: visual.port.name,
            active: false
          };
          this.pollings.push(polling);
        }
      });
    }
  },
  mounted() {
    console.debug("----Mounted Visualization----");
  },
  beforeDestroy() {
    console.debug("Sent cleanup request to FireFlask");
    this.axiosRequest(null, this.paths.cleanup, "POST");
    this.pollings.forEach(polling => {
      console.debug(
        `Clearing interval [${polling.interval}] of ${polling.port}`
      );
      clearInterval(polling.interval);
    });
  },
  methods: {
    sendByte(pin, port) {
      console.debug(`Setting ${pin} on ${port}`);
      var payload = {
        pin,
        port
      };
      console.debug(payload);
      this.axiosRequest(payload, this.paths.shift, "POST");
    },
    axiosRequest(payload, path, request) {
      if (request === "POST") {
        axios
          .post(path, payload)
          .then(response => {
            if (payload) {
              console.debug(
                `${path}? \n Status-Code ${response.status} - ${
                  response.data.status
                } [${payload.pin}]@${payload.port}`
              );
              this.response = response.data;
            } else {
              console.debug(
                `${path}? \n Status-Code ${response.status} - ${
                  response.data.status
                }`
              );
            }
          })
          .catch(error => {
            // Error 😨
            if (error.response) {
              /*
               * The request was made and the server responded with a
               * status code that falls out of the range of 2xx
               */
              console.error(error.response.data);
              console.error(error.response.status);
              console.error(error.response.headers);
            } else if (error.request) {
              /*
               * The request was made but no response was received, `error.request`
               * is an instance of XMLHttpRequest in the browser and an instance
               * of http.ClientRequest in Node.js
               */
              console.error(error.request);
            } else {
              // Something happened in setting up the request and triggered an Error
              console.error("Error", error.message);
            }
            console.error(error.config);
          });
      }
    },
    switchCountdown(ref, id) {
      // console.debug(this.$refs[ref][0])
      this.$refs[ref][0].switchCountdown();
      this.$nextTick(() => {
        this.counters[id - 1].state =
          this.$refs[ref][0].state === "stoped" ? "On" : "Off";
      });
    },
    flattenVisuals(visuals) {
      console.debug("Flattening visuals...");
      const fV = visuals.flatMap(visual => {
        return visual.IO.map((io, pos) => {
          return {
            port: visual.port.name,
            id: visual.port.id,
            boardType: visual.board.boardType,
            ioType: visual.board.rpiType,
            io: io,
            pos: pos,
            state: visual.pinStates[io],
            key: `${visual.board.boardType}-${io}@${visual.port.name} => ${
              visual.pinStates[io]
            }`
          };
        });
      });
      for (var i = 0; i < fV.length; i++) {
        this.flatVisuals[`${fV[i].port}-${fV[i].io}`] = fV[i];
      }
    },
    positionLabel(visual) {
      return {
        height: 100 + "px",
        width: 200 + "px",
        position: "absolute",
        top: (visual.port.id - 1) * 100 + "px",
        left: -210 + "px"
      };
    },
    togglePolling(visual) {
      const index = this.pollings.findIndex(
        polling => polling.id === visual.port.id
      );
      if (this.pollings[index].active) {
        console.debug(
          `Clearing interval [${this.pollings[index].interval}] of ${
            visual.board.boardType
          } @ ${visual.port.name}`
        );
        clearInterval(this.pollings[index].interval);
        this.pollings[index].active = false;
      } else {
        this.pollings[index].interval = setInterval(() => {
          this.sendByte(visual.IO, visual.port.name);
        }, 5000);
        this.pollings[index].active = true;
        console.debug(
          `Created interval [${this.pollings[index].interval}] for ${
            visual.board.boardType
          } @ ${visual.port.name}`
        );
      }
    }
  }
};
</script>

<style lang="scss">
/*** EXAMPLE ***/
#content {
  background-image: url("../assets/visuals_bg.png");
  background-repeat: no-repeat;
  background-size: contain;
  height: 500px;
  width: 1000px;
  margin: auto;
  border: 2px solid black;
  position: relative;
}
.label {
  border: 1px dashed grey;
  position: relative;
}
.pin {
  border: 1px dashed orange;
}
// This is the css for vue-draggable-resizable
// DON'T EDIT unless customization is needed
.vdr {
  touch-action: none;
  position: absolute;
  box-sizing: border-box;
  border: 1px dashed black;
}
.handle {
  box-sizing: border-box;
  position: absolute;
  width: 10px;
  height: 10px;
  background: #eee;
  border: 1px solid #333;
}
.handle-tl {
  top: -10px;
  left: -10px;
  cursor: nw-resize;
}
.handle-tm {
  top: -10px;
  left: 50%;
  margin-left: -5px;
  cursor: n-resize;
}
.handle-tr {
  top: -10px;
  right: -10px;
  cursor: ne-resize;
}
.handle-ml {
  top: 50%;
  margin-top: -5px;
  left: -10px;
  cursor: w-resize;
}
.handle-mr {
  top: 50%;
  margin-top: -5px;
  right: -10px;
  cursor: e-resize;
}
.handle-bl {
  bottom: -10px;
  left: -10px;
  cursor: sw-resize;
}
.handle-bm {
  bottom: -10px;
  left: 50%;
  margin-left: -5px;
  cursor: s-resize;
}
.handle-br {
  bottom: -10px;
  right: -10px;
  cursor: se-resize;
}
@media only screen and (max-width: 768px) {
  [class*="handle-"]:before {
    content: "";
    left: -10px;
    right: -10px;
    bottom: -10px;
    top: -10px;
    position: absolute;
  }
}
</style>
