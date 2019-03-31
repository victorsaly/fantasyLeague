<template>
  <section class="container-fluid">
    <nav class="navbar navbar-light">
     <img src="logo.jpg" alt="">
     <span class="navbar-brand" v-if="current_week > 0">Week 
          <input style="width:40px;border:none;background:none;" type="number" min="1" :max="max_week" v-model="current_week"> {{ events[current_week-1].deadline_time_formatted}}
   </span>
  
     <a class="navbar-brand" href="#" v-if="max_week > 1" v-on:click="playWeeklyPointChart(1)">
<fa :icon="['fas', 'play']" />
       Run Weekly Chart</a>
   
      <a class="navbar-brand" href="#">{{league.name}} League</a>
   </nav>
    <div class="row" style="height: calc(100vh - 162px);padding:20px;">
      <div v-if="current_week == 0" class="loading">Loading&#8230;</div>

       <div class="col-md-6">
        <table class="table" v-if="current_week > 0">
          <thead>
            <tr>
              <th>
                Graph
              </th>
            </tr>
          </thead>
          <tbody>
            <tr>
              <td style="padding:0">
                <transition-group name="flip-list" class="list-group" tag="ul">
                    <li v-for="(user, index) in orderedUsers" v-bind:key="user.id" class="list-group-item">
                      <div style="width: 100% !important; ">
                        <div
                          class="bar"
                          :style="'background:#' + colours[index] + ';width: calc(' + ((100 * user.total)/orderedUsers[0].total) + '%' + ' - 45px)'"
                        >{{index + 1}} - {{ user.name }}</div>
                        <div style="float:left; width:40px; line-height:45px;">
                          <animated-number :value="user.total" :formatValue="formatToInteger" :duration="500"/>
                        </div>
                      </div>
                    </li>
                  </transition-group>
                </td>
            </tr>
          </tbody>
      </table>
      </div>
      <div class="col-md-6">
        <table class="table" v-if="current_week > 0">
          <thead>
            <tr>
              <th></th>
              <th>Position</th>
              <th>Name</th>
              <th>Points</th>
              <th>Total</th>
              <th style="text-align:right">Diff/ T.Diff</th>
              <!-- <th>Week</th> -->
            </tr>
          </thead>
          <tbody>
            <tr v-for="(user, index) in orderedUsers" v-bind:key="user.id">
              <td :style="'background:#' + colours[user.index] + ';color:white;'">
                <fa :icon="move(user.movement)" />
              </td>
              <td :style="'background:#' + colours[user.index] + ';color:white;'">{{index+1}}</td>
              <td :style="'background:#' + colours[user.index] + ';color:white;'">{{user.name}}</td>
              <td :style="'background:#' + colours[user.index] + ';color:white;'">
                  <animated-number :value="user.score" :formatValue="formatToInteger" :duration="500"/>
              </td>
              <td :style="'background:#' + colours[user.index] + ';color:white;'">
                  <animated-number :value="user.total" :formatValue="formatToInteger" :duration="500"/>
              </td>
              <td :style="'background:#' + colours[user.index] + ';color:white;text-align:right'">
                {{user.total - (index-1 >=0 ? orderedUsers[index-1].total : user.total )}} / 
                {{user.total - (index-1 >=0 ? orderedUsers[0].total : user.total )}}
              </td>
              <!-- <td>
                  <animated-number :value="current_week" :formatValue="formatToInteger" :duration="500"/>  
              </td> -->
            </tr>
          </tbody>
        </table>
      </div>
     
      <!-- <div class="col-sm-12">
        <button v-if="max_week > 1" v-on:click="playWeeklyPointChart(1)">Play Weekly Chart</button>
      </div> -->
    </div>
    <div>
    </div>
    
  </section>


</template>

<script>
import { fas } from '@fortawesome/free-solid-svg-icons'
import axios from "axios";
import _ from "lodash";
let interval = null;
export default {
  validate ({ params }) {
    // Must be a number
    return /^\d+$/.test(params.id)
  },
  data: () => ({
    league:{},
    events:[],
    max_week:0,
    current_week: 0,
    users: [],
    colours:[
      'C01602',
      'AE2106',
      '9D2C0B',
      '8B3810',
      '7A4315',
      '684F1A',
      '575A1E',
      '456523',
      '347128',
      '227C2D',
      '118832'
    ]
    
  }),
  created(){
    this.loadData();
  },
  computed: {
    orderedUsers: function() {
      return _.orderBy(this.users, "total", "desc");
    },
    
  },
  watch:{
    current_week: function (val) {
      let ce = val -1;
      if (this.users[0]!=undefined && this.users[0].history[ce] != undefined){
        this.users.forEach((u)=>{
          u.total = u.history[ce].total_points;
          u.score =  u.history[ce].points;
          u.movement = u.history[ce].movement;
        });
      }

      if (val>=this.max_week && interval != null){
        
        clearInterval(interval);
        interval = null;
      }
    }
  },
  methods: {
    move (movement){
      switch(movement){
        case "same":
          return fas.faAngleRight;
          break;
        case "up":
          return fas.faAngleUp;
          breakl
        case "down":
          return fas.faAngleDown;
          break;
      }
      return fas;
    },
    loadData() {
      let self = this;

      axios.get("drf/events/").then(function(response){
        self.events = response.data;
      });

      axios.get("drf/leagues-classic-standings/" + this.$route.params.id).then(function(data) {

        var items = [];
        var users = data.data != undefined ? data.data.standings.results : "";
        self.league = data.data.league;
        users.forEach((user, i) => {
          items.push({
            index : i,
            id: user.entry,
            movement: user.movement,
            score: user.event_total,
            total: user.total,
            color: "#" + (Math.random()*0xFFFFFF<<0).toString(16),
            name: user.player_name,
            picks: [],
            history:[],
            chips:[]
          });
        });
        let current_week = 1;
        self.users = items;
        axios.get("drf/entry/" + items[0].id).then(function(d) {
          self.current_week = d.data.entry.current_event;
          self.max_week = self.current_week;
           // Create all the promises required to bring the week users data
            let promises = [];
            self.users.forEach(user => {
              promises.push(
                axios.get(
                  "drf/entry/" + user.id + "/history"
                )
              );
            });

            // Exec promise all then get data and update the picks
            Promise.all(promises).then(data => {
              data.forEach((d, i) => {
                self.users[i].history = d.data.history;
                self.users[i].chips = d.data.chips;
              });
            });

         
        });
      });
    },
    shuffle: function() {
      this.users.forEach(element => {
        element.total = _.random(1, 100);
      });
      this.users = _.shuffle(this.users);
    },
    formatToInteger(value) {
      return `${value.toFixed(0)}`;
    },
    playWeeklyPointChart(start){
      var self = this;
      this.current_week=start;
      interval = setInterval(function() {
            if (self.current_week<self.max_week) {
                self.current_week++;
            }
        }, 600);
    }
  }
};
</script>

<style>
.table thead th {
    vertical-align: bottom;
    border-bottom: 2px solid #dee2e6;
    background: #37003c;
    color: white;
}
.navbar{
  background: #47fd86;
      margin: 0 -15px;
}
.flip-list-move {
  transition: transform 1s;
}
.bar {
  transition: width 1s;
  float: left;
  height: 47px;
  padding-left: 5px;
  background: blue;
  text-align: left;
  color: white; 
  line-height: 45px;
}
.container {
  margin: 0 auto;
  min-height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
  text-align: center;
}

.title {
  font-family: "Quicksand", "Source Sans Pro", -apple-system, BlinkMacSystemFont,
    "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif;
  display: block;
  font-weight: 300;
  font-size: 100px;
  color: #35495e;
  letter-spacing: 1px;
}

.subtitle {
  font-weight: 300;
  font-size: 42px;
  color: #526488;
  word-spacing: 5px;
  padding-bottom: 15px;
}

.links {
  padding-top: 15px;
}
.list-group-item{
  border: none;
  /* padding: 0.65rem 1.25rem; */
  padding: 0 0  2px 0px;
}

/* Absolute Center Spinner */
.loading {
  position: fixed;
  z-index: 999;
  height: 2em;
  width: 2em;
  overflow: visible;
  margin: auto;
  top: 0;
  left: 0;
  bottom: 0;
  right: 0;
}

/* Transparent Overlay */
.loading:before {
  content: '';
  display: block;
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  /* background-color: rgba(0,0,0,0.3); */
}

/* :not(:required) hides these rules from IE9 and below */
.loading:not(:required) {
  /* hide "loading..." text */
  font: 0/0 a;
  color: transparent;
  text-shadow: none;
  background-color: transparent;
  border: 0;
}

.loading:not(:required):after {
  content: '';
  display: block;
  font-size: 10px;
  width: 1em;
  height: 1em;
  margin-top: -0.5em;
  -webkit-animation: spinner 1500ms infinite linear;
  -moz-animation: spinner 1500ms infinite linear;
  -ms-animation: spinner 1500ms infinite linear;
  -o-animation: spinner 1500ms infinite linear;
  animation: spinner 1500ms infinite linear;
  border-radius: 0.5em;
  -webkit-box-shadow: rgba(0, 0, 0, 0.75) 1.5em 0 0 0, rgba(0, 0, 0, 0.75) 1.1em 1.1em 0 0, rgba(0, 0, 0, 0.75) 0 1.5em 0 0, rgba(0, 0, 0, 0.75) -1.1em 1.1em 0 0, rgba(0, 0, 0, 0.5) -1.5em 0 0 0, rgba(0, 0, 0, 0.5) -1.1em -1.1em 0 0, rgba(0, 0, 0, 0.75) 0 -1.5em 0 0, rgba(0, 0, 0, 0.75) 1.1em -1.1em 0 0;
  box-shadow: rgba(0, 0, 0, 0.75) 1.5em 0 0 0, rgba(0, 0, 0, 0.75) 1.1em 1.1em 0 0, rgba(0, 0, 0, 0.75) 0 1.5em 0 0, rgba(0, 0, 0, 0.75) -1.1em 1.1em 0 0, rgba(0, 0, 0, 0.75) -1.5em 0 0 0, rgba(0, 0, 0, 0.75) -1.1em -1.1em 0 0, rgba(0, 0, 0, 0.75) 0 -1.5em 0 0, rgba(0, 0, 0, 0.75) 1.1em -1.1em 0 0;
}

/* Animation */

@-webkit-keyframes spinner {
  0% {
    -webkit-transform: rotate(0deg);
    -moz-transform: rotate(0deg);
    -ms-transform: rotate(0deg);
    -o-transform: rotate(0deg);
    transform: rotate(0deg);
  }
  100% {
    -webkit-transform: rotate(360deg);
    -moz-transform: rotate(360deg);
    -ms-transform: rotate(360deg);
    -o-transform: rotate(360deg);
    transform: rotate(360deg);
  }
}
@-moz-keyframes spinner {
  0% {
    -webkit-transform: rotate(0deg);
    -moz-transform: rotate(0deg);
    -ms-transform: rotate(0deg);
    -o-transform: rotate(0deg);
    transform: rotate(0deg);
  }
  100% {
    -webkit-transform: rotate(360deg);
    -moz-transform: rotate(360deg);
    -ms-transform: rotate(360deg);
    -o-transform: rotate(360deg);
    transform: rotate(360deg);
  }
}
@-o-keyframes spinner {
  0% {
    -webkit-transform: rotate(0deg);
    -moz-transform: rotate(0deg);
    -ms-transform: rotate(0deg);
    -o-transform: rotate(0deg);
    transform: rotate(0deg);
  }
  100% {
    -webkit-transform: rotate(360deg);
    -moz-transform: rotate(360deg);
    -ms-transform: rotate(360deg);
    -o-transform: rotate(360deg);
    transform: rotate(360deg);
  }
}
@keyframes spinner {
  0% {
    -webkit-transform: rotate(0deg);
    -moz-transform: rotate(0deg);
    -ms-transform: rotate(0deg);
    -o-transform: rotate(0deg);
    transform: rotate(0deg);
  }
  100% {
    -webkit-transform: rotate(360deg);
    -moz-transform: rotate(360deg);
    -ms-transform: rotate(360deg);
    -o-transform: rotate(360deg);
    transform: rotate(360deg);
  }
}
</style>
