<template>
  <div class="container text-center mt-5">
    <div class="versus-bg" v-show="versusShow">
    </div>

    <div v-if="(table != null)" class="row align-items-center mb-3 justify-content-evenly border-bottom">
      <div class="col-3">
        <div class="fs-1 fw-semibold">{{ teams[0].name}}</div>
      </div>
      <div class="col-4">
        <div class="fs-1 fw-semibold">Задания</div>
      </div>
      <div class="col-3">
        <div class="fs-1 fw-semibold">{{ teams[1].name}}</div>
      </div>
    </div>

    <div v-if="(table != null)">
      <div v-for="item in table" :key="item.name" class="row align-items-center pb-3 justify-content-evenly">
        <div :id="'members'+item[0].id" :class="[
          {'text-bg-success': item[0].is_winner == '1'},
          {'text-bg-danger': item[2].is_winner == '1'},
          {'versus first': item[0].id == '1'},
          'card col-3 d-flex flex-row p-2']">
          <img class="rounded" :src="item[0].photo" height="100" :alt="item[0].name">
          <div class="container-fluid align-self-lg-center fs-5 d-flex flex-column justify-content-center fw-semibold">
            <div>{{ item[0].last_name }}</div>
            <div>{{ item[0].name }}</div>
          </div>
        </div>
        <div :class="[{'text-bg-success': item[1].is_solved == '1'}, 'card col-4 p-1']">
          <div class="fs-5 fw-semibold">{{ item[1].name }}</div>
        </div>
        <div :id="'members'+item[0].id" :class="[
          {'text-bg-success': item[2].is_winner == '1'},
          {'text-bg-danger': item[0].is_winner == '1'},
          {'versus second': item[2].id == '2'},
          'card col-3 d-flex flex-row p-2']">
          <img class="rounded" :src="item[2].photo" height="100" :alt="item[2].name">
          <div class="container-fluid align-self-lg-center fs-5 d-flex flex-column justify-content-center fw-semibold">
            <div>{{ item[2].last_name }}</div>
            <div>{{ item[2].name }}</div>
          </div>
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
      title: 'CTF Final',
      versusShow: true,
      members: [],
      tasks: [],
      teams: [],
    }
  },
  created() {
    document.title = this.title;
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
        // this.showVersusWinner(data.tasks, data.members);
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
    },
    getSolvedTaskGroup(tasks, members) {
      var solved_task = _.find(tasks, task => {
        return task.is_solved == '1' 
          && _.find(this.tasks, item => {
            return item.id == task.id && item.is_solved !== task.is_solved;
          }) != undefined;
      });

      if (solved_task == undefined) {
        return null;
      }

      var membersTask = _.filter(members, {'task_id': solved_task.id});

      return {
        task: solved_task,
        members: membersTask,
      };
    },
    showVersusWinner(tasks, members){
      var versusGroup = this.getSolvedTaskGroup(tasks, members);
      if (versusGroup != null) {
        var element_member1 = document.getElementById('members' + versusGroup.members[0].id);
        var element_member2 = document.getElementById('members' + versusGroup.members[1].id);

        element_member1.classList.add("winner");
        element_member2.classList.add("loser");
      }
    }
  }
}
</script>

<style>
.versus-bg {
  z-index: 1000;
  position: fixed;
  top:0;
  left:0;
  width:100vw;
  height:100vh;
  background-color: rgba(34, 34, 34, 0.744);
}

.versus {
  z-index: 1100;
  animation-duration: 2.5s;
  animation-timing-function: easy-in-out;
  animation-direction: alternate;
}

.versus.first {
  animation-name: versus-first;
}

.versus.second {
  animation-name: versus-second;
}

@keyframes versus-first {
  0% {transform: scale(1);}
  50% {
    transform: scale(1.5) translate(5em,8em);
    background-color: #fff;
  }
  55% {
    transform: scale(1.5) translate(5em,8em);
    background-color: green;
  }
  100% {
    transform: scale(1);
    background-color: green;
  }
}

@keyframes versus-second {
  0% {transform: scale(1);}
  80% {
    transform: scale(1.5) translate(-5em,8em);
  }
}

.winner {

}

.loser {

}
</style>
