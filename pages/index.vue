<template>
  <section class="container-fluid">
    <div class="row">
      <div class="col-sm-12">
        <h2>
          {{league.name}} League
        </h2>
        <h3 v-if="current_week > 0">Week 
          <input style="width:60px;border:none;" type="number" min="1" :max="max_week" v-model="current_week"> {{ events[current_week-1].deadline_time_formatted}}
        </h3>
      </div>
      <div class="col-sm-6">
        <table class="table" v-if="current_week > 0">
          <thead>
            <tr>
              <th></th>
              <th>Position</th>
              <th>Name</th>
              <th>Points</th>
              <th>Total</th>
              <th style="text-align:right">Diff/ T.Diff</th>
              <th>Week</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(user, index) in orderedUsers" v-bind:key="user.id">
              <td :style="'background:#' + colours[user.index] + ';'">
                <!-- <font-awesome-icon :icon="fas.faAngleRight" /> -->
                <fa :icon="['fas', 'dollar-sign']" />

              </td>
              <td>{{index+1}}</td>
              <td>{{user.name}}</td>
              <td>
                  <animated-number :value="user.score" :formatValue="formatToInteger" :duration="500"/>
              </td>
              <td>
                  <animated-number :value="user.total" :formatValue="formatToInteger" :duration="500"/>
              </td>
              <td style="text-align:right">
                {{user.total - (index-1 >=0 ? orderedUsers[index-1].total : user.total )}} / 
                {{user.total - (index-1 >=0 ? orderedUsers[0].total : user.total )}}
              </td>
              <td>
                  <animated-number :value="current_week" :formatValue="formatToInteger" :duration="500"/>  
              </td>
            </tr>
          </tbody>
        </table>
      </div>
      <div class="col-sm-6">
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
              <td>
                <transition-group name="flip-list" class="list-group" tag="ul">
                    <li v-for="(user, index) in orderedUsers" v-bind:key="user.id" class="list-group-item">
                      <div style="width: 100% !important; ">
                        <div
                          class="bar"
                          :style="'background:#' + colours[index] + ';width: calc(' + ((100 * user.total)/orderedUsers[0].total) + '%' + ' - 45px)'"
                        >{{index + 1}} - {{ user.name }}</div>
                        <div style="float:left; width:40px;">
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
      <div class="col-sm-12">
        <button v-if="max_week > 1" v-on:click="playWeeklyPointChart(1)">Play Weekly Chart</button>
      </div>
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
        });
      }

      if (val>=this.max_week && interval != null){
        
        clearInterval(interval);
        interval = null;
      }
    }
  },
  methods: {
    fas (movement){
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

      axios.get("drf/leagues-classic-standings/724060").then(function(data) {

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
            history:[]
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
                //self.users[i].score = self.users[i].history[self.users[i].history.lenght].points;
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
.flip-list-move {
  transition: transform 1s;
}
.bar {
  transition: width 1s;
  float: left;
  height: 25px;
  padding-left: 5px;
  background: blue;
  text-align: left;
  color: white;
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
  padding: 0.65rem 1.25rem;
}
</style>
