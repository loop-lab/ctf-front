<template>
  <div class="container text-center">
    <h1>CTF</h1>
    <div v-if="(table != null)" class="row align-items-start pb-3 section">
      <div class="col">
        {{ teams[0].name}}
      </div>
      <div class="col">
        Задания
      </div>
      <div class="col">
        {{ teams[1].name}}
      </div>
    </div>

    <div v-if="(table != null)">
      <div v-for="item in table" :key="item.name" class="row align-items-center pb-3 section">
        <div class="col">
          <img :src="item[0].photo" class="img-thumbnail" :alt="item[0].name">
          {{ item[0].last_name }} {{ item[0].name }}
        </div>
        <div class="col">
          {{ item[1].name }}
        </div>
        <div class="col">
          <img :src="item[2].photo" class="img-thumbnail" :alt="item[2].name">
          {{ item[2].last_name }} {{ item[2].name }}
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import _ from 'lodash';
import Pusher from 'pusher-js';
import { app } from './main.js'

const axios = require('axios').default;

export default {
  name: 'App',
  data() {
    return {
      members: [],
      tasks: [],
      teams: [],
    }
  },
  created() {
    axios.get('http://37.230.113.138/api/data')
      .then(response => {
        this.table = response.data;
      });
    this.subscribeToBack();
  },
  computed: {
    table: {
      cache: false,
      get() {
        if (this.members.length == 0 || this.tasks == 0 || this.teams == 0 ) {
          return null;
        }

        var members1 = [];
        var members2 = [];

        _.each(this.members, function(member) {
          if (member.team_id == 1) {
            members1.push(member);
          }

          if (member.team_id == 2) {
            members2.push(member);
          }
        });

        _.sortBy(this.tasks, 'id');
        _.sortBy(members1, 'task_id');
        _.sortBy(members2, 'task_id');

        var data = [];

        for (var i = 0; i < this.tasks.length; i++) {
          data.push([
            members1[i],
            this.tasks[i],
            members2[i]
          ]);
        }

        return data;
      },
      set(data) {
        this.members = data.members;
        this.tasks = data.tasks;
        this.teams = data.teams;
      }
    }
  },
  methods: {
    subscribeToBack() {
      Pusher.logToConsole = false;
      var pusher = new Pusher('3a1d3cf530425138eade', {
        cluster: 'eu'
      });
      var channel = pusher.subscribe('tasks');
      channel.bind('App\\Events\\StatusTasksUpdated', function(response) {
        app.table = response.data;
      });
    }
  }
}
</script>

<style>
.section {
  border: solid;
}
</style>
