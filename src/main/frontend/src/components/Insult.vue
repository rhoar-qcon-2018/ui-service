<template>
  <div>
    <div class="row inline justify-center items-start">
      <div class="col-12">
        <q-toggle class="short" right-label v-model="isReactiveEnabled" color="yellow" :label="connStatus"></q-toggle>
      </div>
    </div>
    <div class="row inline justify-center items-start">
      <div class="col-12" style="padding-right: 4px;">
        <div class="side-by-side">
          <q-btn color="red" @click="getInsult">Insult {{target}}!</q-btn>
          <q-btn color="green" @click="clearHistory">I Didn't Mean It!</q-btn>
        </div>
      </div>
      <div class="col-6" style="padding-left: 4px;">
        <h3>My Insults</h3>
      </div>
      <div class="col-6" style="padding-left: 4px;">
        <h3>Favorites</h3>
      </div>
      <div class="col-6" style="padding-left: 4px;">
        <q-card v-for="insult in insults" color="primary">
          <q-card-title text-color="black">
            Thou dost be a {{ insult.adj1 }}, {{ insult.adj2 }}, {{ insult.noun }}
            <q-btn fab flat color="transparent" text-color="primary" style="width: 32px; height: 32px; top: 0; left: 48px" icon="thumb_up" @click="favoriteInsult(insult)" />
          </q-card-title>
        </q-card>
      </div>
      <div class="col-6" style="padding-left: 4px;">
        <q-card v-for="insult in favorites">
          <q-card-title text-color="black">
            Thou dost be a {{ insult.adj1 }}, {{ insult.adj2 }}, {{ insult.noun }}
          </q-card-title>
        </q-card>
      </div>
    </div>
  </div>
</template>

<script>
import { QBtn, QToggle, QInput, QCard, QCardActions, QCardMain, QIcon } from 'quasar'
import 'quasar-extras/material-icons'
import EventBus from 'vertx3-eventbus-client'
import InsultService from 'assets/insult_service-proxy'
import axios from 'axios'

export default {
  name: 'insult',
  components: {
    QBtn,
    QToggle,
    QInput,
    QCard,
    QCardMain,
    QCardActions,
    QIcon
  },
  data() {
    return {
        nameInput: "",
        isReactiveEnabled: false,
        eventBus: {},
        service: {},
        rest: {},
        baseURL: "",
        insults: [],
        favorites: []
    }
  },
  computed: {
      target() {
          if (this.nameInput.length == 0) {
              return "ME";
          } else {
              return this.nameInput;
          }
      },
      connStatus() {
          if (this.isReactiveEnabled) {
              return "WebSockets";
          } else {
              return "REST";
          }
      }
  },
  methods: {
    clearHistory() {
        this.insults = [];
    },
    favoriteInsult(insult) {
      var resultHandler = (err, res) => {
          if (err!==null) {
              console.log("Error calling service: "+err);
          }
      }
      this.service.publish(insult, resultHandler);
    },
    getInsult() {
      if (this.isReactiveEnabled) {
          var resultHandler = (err, res) => {
              if (err===null) {
                  if (this.insults.length == 10) {
                      this.insults.shift();
                  }
                  this.insults.push(res);
              } else {
                  console.log("Error calling service: "+err);
              }
          };
          this.service.getREST(resultHandler);
      } else {
          var reqPromise = {};
          reqPromise = this.rest.get("/api/v1/insult");
          reqPromise
              .then((resp) => {
                  if (this.insults.length == 10) {
                      this.insults.shift();
                  }
                  this.insults.push(formatter(resp.data));
              })
              .catch((err) => {
                  console.log(err);
              });
      }
    }
  },
  created: function () {
    let loc_proto = window.location.protocol;
    let loc_host = window.location.hostname.replace(/^ui-service/, 'insult-service');
    let loc_port = window.location.port;

    if (loc_port === "") {
        this.baseURL = loc_proto + "//" + loc_host;
    } else {
        this.baseURL = loc_proto + "//" + loc_host + ":" + loc_port;
    }
    console.log("BaseURL: "+this.baseURL);

    var options = {
        vertxbus_reconnect_attempts_max: Infinity, // Max reconnect attempts
        vertxbus_reconnect_delay_min: 1000, // Initial delay (in ms) before first reconnect attempt
        vertxbus_reconnect_delay_max: 10000, // Max delay (in ms) between reconnect attempts
        vertxbus_reconnect_exponent: 2, // Exponential backoff factor
        vertxbus_randomization_factor: 0.5 // Randomization factor between 0 and 1
    };

    this.eventBus = new EventBus(this.baseURL+"/eventbus", options);
    this.eventBus.enableReconnect(true);

    this.eventBus.onopen = () => {
        console.log("BaseURL: "+this.baseURL);
        this.service = new InsultService(this.eventBus, "insult.service");

        this.eventBus.registerHandler('insult.favorites', (err, res) => {
            if (err===null) {
                if (this.favorites.length == 10) {
                    this.favorites.shift();
                }
                if (typeof res === '')
                this.favorites.push(res);
            }
        });
    };
    this.rest = axios.create({
        baseURL: this.baseURL,
        timeout: 1000
    });
  },
  destroyed: function () {
    this.eventBus.close();
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style lang="stylus" scoped>
.row
  width: 100%
  margin-top: 4px
.col-6
  width: 50%
  margin: auto
  text-align center
.side-by-side
  justify-content
.col-12
  width: 100%
  margin: auto
  text-align center
.short
  width: 120px
  margin: auto
</style>
