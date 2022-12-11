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
      <div v-for="(item, index) in table" :key="item.name" class="row align-items-center pb-3 justify-content-evenly">
        <div :id="'members'+item[0].id" ref="leftMembers" :class="[
          'card col-3 d-flex flex-row p-2',
          'item' + (index*2 + 1),
          getWinnerClass(item, 0, 2),
        ]">
          <img class="rounded" :src="item[0].photo" height="100" :alt="item[0].name">
          <div class="container-fluid align-self-lg-center fs-5 d-flex flex-column justify-content-center fw-semibold">
            <div>{{ item[0].last_name }}</div>
            <div>{{ item[0].name }}</div>
          </div>
        </div>
        <div :class="[{'text-bg-success': item[1].is_solved == '1'}, 'card col-4 p-1']">
          <div class="fs-5 fw-semibold">{{ item[1].name }}</div>
        </div>
        <div :id="'members'+item[2].id" ref="rightMembers" :class="[
          'card col-3 d-flex flex-row p-2',
          'item' + (index*2 + 2),
          getWinnerClass(item, 2, 0),
        ]">
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
      firstGetTable: false,
      title: 'CTF Final',
      versusShow: false,
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
    setTimeout(() => {
      this.created = true;
    }, 1000)
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
        this.showVersusWinner(data.tasks, data.members);
        if (this.versusShow == true) {
          setTimeout(() => {
            this.members = data.members;
            this.tasks = data.tasks;
            this.teams = data.teams;
          }, 5000)
        } else {
          this.members = data.members;
          this.tasks = data.tasks;
          this.teams = data.teams;
        }
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
            return item.id == task.id && Number(item.is_solved) !== Number(task.is_solved);
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
        this.versusShow = true;
        element_member1.classList.add('versus');
        element_member1.classList.add('left');
        element_member2.classList.add('versus');
        element_member2.classList.add('right');
        setTimeout(() => {
          element_member1.classList.add('out');
          element_member2.classList.add('out');
        }, 2500)
        setTimeout(() => {
          element_member1.classList.add(versusGroup.members[0].is_winner == '1' ? 'winner' : 'loser');
          element_member2.classList.add(versusGroup.members[1].is_winner == '1' ? 'winner' : 'loser');
        }, 2499)
        setTimeout(() => {
          element_member1.classList.remove("versus");
          element_member2.classList.remove("versus");
          element_member1.classList.remove("left");
          element_member2.classList.remove("right");
          element_member1.classList.remove("out");
          element_member2.classList.remove("out");
          this.versusShow = false;
        }, 5000);
      }
    },
    getWinnerClass(item, id, versus_id) {
      this.firstGetTable = true;
      if (this.firstGetTable) {
        if (item[id].is_winner == '1' && item[versus_id].is_winner == '0' && item[1].is_solved == '1') {
          return 'winner';
        }

        if (item[id].is_winner == '0' && item[versus_id].is_winner == '1' && item[1].is_solved == '1') {
          return 'loser';
        }
      }
      return '';
    }
  }
}
</script>

<style>
.loser {
  background: red;
}
.winner {
  background: green;
}
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
}

.item1.versus.left {
  animation-name: versus-left-1;
  transform: scale(1.5) translate(5em,12em);
}
.item3.versus.left {
  animation-name: versus-left-2;
  transform: scale(1.5) translate(5em,6em);
}
.item5.versus.left {
  animation-name: versus-left-3;
  transform: scale(1.5) translate(5em,0em);
}
.item7.versus.left {
  animation-name: versus-left-4;
  transform: scale(1.5) translate(5em,-6em);
}
.item9.versus.left {
  animation-name: versus-left-5;
  transform: scale(1.5) translate(5em,-12em);
}
.item11.versus.left {
  animation-name: versus-left-6;
  transform: scale(1.5) translate(5em,-18em);
}
.item13.versus.left {
  animation-name: versus-left-7;
  transform: scale(1.5) translate(5em,-24em);
}

@keyframes versus-left-1 {
  0% {transform: scale(1);}
  50% {
    transform: scale(1.5) translate(5em,12em);
  }
  55% {
    transform: scale(1.5) translate(5em,12em);
  }
  100% {
    transform: scale(1.5) translate(5em,12em);
  }
}
@keyframes versus-left-2 {
  0% {transform: scale(1);}
  50% {
    transform: scale(1.5) translate(5em,6em);
  }
  55% {
    transform: scale(1.5) translate(5em,6em);
  }
  100% {
    transform: scale(1.5) translate(5em,6em);
  }
}
@keyframes versus-left-3 {
  0% {transform: scale(1);}
  50% {
    transform: scale(1.5) translate(5em,0em);
  }
  55% {
    transform: scale(1.5) translate(5em,0em);
  }
  100% {
    transform: scale(1.5) translate(5em,0em);
  }
}
@keyframes versus-left-4 {
  0% {transform: scale(1);}
  50% {
    transform: scale(1.5) translate(5em,-6em);
  }
  55% {
    transform: scale(1.5) translate(5em,-6em);
  }
  100% {
    transform: scale(1.5) translate(5em,-6em);
  }
}
@keyframes versus-left-5 {
  0% {transform: scale(1);}
  50% {
    transform: scale(1.5) translate(5em,-12em);
  }
  55% {
    transform: scale(1.5) translate(5em,-12em);
  }
  100% {
    transform: scale(1.5) translate(5em,-12em);
  }
}
@keyframes versus-left-6 {
  0% {transform: scale(1);}
  50% {
    transform: scale(1.5) translate(5em,-18em);
  }
  55% {
    transform: scale(1.5) translate(5em,-18em);
  }
  100% {
    transform: scale(1.5) translate(5em,-18em);
  }
}
@keyframes versus-left-7 {
  0% {transform: scale(1);}
  50% {
    transform: scale(1.5) translate(5em,-24em);
  }
  55% {
    transform: scale(1.5) translate(5em,-24em);
  }
  100% {
    transform: scale(1.5) translate(5em,-24em);
  }
}

.item2.versus.right {
  animation-name: versus-right-1;
  transform: scale(1.5) translate(-5em,12em);
}
.item4.versus.right {
  animation-name: versus-right-2;
  transform: scale(1.5) translate(-5em,6em);
}
.item6.versus.right {
  animation-name: versus-right-3;
  transform: scale(1.5) translate(-5em,0em);
}
.item8.versus.right {
  animation-name: versus-right-4;
  transform: scale(1.5) translate(-5em,-6em);
}
.item10.versus.right {
  animation-name: versus-right-5;
  transform: scale(1.5) translate(-5em,-12em);
}
.item12.versus.right {
  animation-name: versus-right-6;
  transform: scale(1.5) translate(-5em,-18em);
}
.item14.versus.right {
  animation-name: versus-right-7;
  transform: scale(1.5) translate(-5em,-24em);
}
@keyframes versus-right-1 {
  0% {transform: scale(1);}
  50% {
    transform: scale(1.5) translate(-5em,12em);
  }
  55% {
    transform: scale(1.5) translate(-5em,12em);
  }
  100% {
    transform: scale(1.5) translate(-5em,12em);
  }
}
@keyframes versus-right-2 {
  0% {transform: scale(1);}
  50% {
    transform: scale(1.5) translate(-5em,6em);
  }
  55% {
    transform: scale(1.5) translate(-5em,6em);
  }
  100% {
    transform: scale(1.5) translate(-5em,6em);
  }
}
@keyframes versus-right-3 {
  0% {transform: scale(1);}
  50% {
    transform: scale(1.5) translate(-5em,0em);
  }
  55% {
    transform: scale(1.5) translate(-5em,0em);
  }
  100% {
    transform: scale(1.5) translate(-5em,0em);
  }
}
@keyframes versus-right-4 {
  0% {transform: scale(1);}
  50% {
    transform: scale(1.5) translate(-5em,-6em);
  }
  55% {
    transform: scale(1.5) translate(-5em,-6em);
  }
  100% {
    transform: scale(1.5) translate(-5em,-6em);
  }
}
@keyframes versus-right-5 {
  0% {transform: scale(1);}
  50% {
    transform: scale(1.5) translate(-5em,-12em);
  }
  55% {
    transform: scale(1.5) translate(-5em,-12em);
  }
  100% {
    transform: scale(1.5) translate(-5em,-12em);
  }
}
@keyframes versus-right-6 {
  0% {transform: scale(1);}
  50% {
    transform: scale(1.5) translate(-5em,-18em);
  }
  55% {
    transform: scale(1.5) translate(-5em,-18em);
  }
  100% {
    transform: scale(1.5) translate(-5em,-18em);
  }
}
@keyframes versus-right-7 {
  0% {transform: scale(1);}
  50% {
    transform: scale(1.5) translate(-5em,-24em);
  }
  55% {
    transform: scale(1.5) translate(-5em,-24em);
  }
  100% {
    transform: scale(1.5) translate(-5em,-24em);
  }
}


.item1.versus.left.out {
  animation-name: versus-left-out-1;
  transform: scale(1) translate(0);
}
.item3.versus.left.out {
  animation-name: versus-left-out-2;
  transform: scale(1) translate(0);
}
.item5.versus.left.out {
  animation-name: versus-left-out-3;
  transform: scale(1) translate(0);
}
.item7.versus.left.out {
  animation-name: versus-left-out-4;
  transform: scale(1) translate(0);
}
.item9.versus.left.out {
  animation-name: versus-left-out-5;
  transform: scale(1) translate(0);
}
.item11.versus.left.out {
  animation-name: versus-left-out-6;
  transform: scale(1) translate(0);
}
.item13.versus.left.out {
  animation-name: versus-left-out-7;
  transform: scale(1) translate(0);
}

@keyframes versus-left-out-1 {
  0% {transform: scale(1.5) translate(5em,12em);}
  100% {transform: scale(1) translate(0,0);}
}
@keyframes versus-left-out-2 {
  0% {transform: scale(1.5) translate(5em,6em);}
  100% {transform: scale(1) translate(0,0);}
}
@keyframes versus-left-out-3 {
  0% {transform: scale(1.5) translate(5em,0em);}
  100% {transform: scale(1) translate(0,0);}
}
@keyframes versus-left-out-4 {
  0% {transform: scale(1.5) translate(5em,-6em);}
  100% {transform: scale(1) translate(0,0);}
}
@keyframes versus-left-out-5 {
  0% {transform: scale(1.5) translate(5em,-12em);}
  100% {transform: scale(1) translate(0,0);}
}
@keyframes versus-left-out-6 {
  0% {transform: scale(1.5) translate(5em,-18em);}
  100% {transform: scale(1) translate(0,0);}
}
@keyframes versus-left-out-7 {
  0% {transform: scale(1.5) translate(5em,-24em);}
  100% {transform: scale(1) translate(0,0);}
}

.item2.versus.right.out {
  animation-name: versus-right-out-1;
  transform: scale(1) translate(0,0);
}
.item4.versus.right.out {
  animation-name: versus-right-out-2;
  transform: scale(1) translate(0,0);
}
.item6.versus.right.out {
  animation-name: versus-right-out-3;
  transform: scale(1) translate(0,0);
}
.item8.versus.right.out {
  animation-name: versus-right-out-4;
  transform: scale(1) translate(0,0);
}
.item10.versus.right.out {
  animation-name: versus-right-out-5;
  transform: scale(1) translate(0,0);
}
.item12.versus.right.out {
  animation-name: versus-right-out-6;
  transform: scale(1) translate(0,0);
}
.item14.versus.right.out {
  animation-name: versus-right-out-7;
  transform: scale(1) translate(0,0);
}

@keyframes versus-right-out-1 {
  0% {transform: scale(1.5) translate(-5em,12em);}
  100% {transform: scale(1) translate(0,0)}
}
@keyframes versus-right-out-2 {
  0% {transform: scale(1.5) translate(-5em,6em);}
  100% {transform: scale(1) translate(0,0)}
}
@keyframes versus-right-out-3 {
  0% {transform: scale(1.5) translate(-5em,0em);}
  100% {transform: scale(1) translate(0,0)}
}
@keyframes versus-right-out-4 {
  0% {transform: scale(1.5) translate(-5em,-6em);}
  100% {transform: scale(1) translate(0,0)}
}
@keyframes versus-right-out-5 {
  0% {transform: scale(1.5) translate(-5em,-12em);}
  100% {transform: scale(1) translate(0,0)}
}
@keyframes versus-right-out-6 {
  0% {transform: scale(1.5) translate(-5em,-18em);}
  100% {transform: scale(1) translate(0,0)}
}
@keyframes versus-right-out-7 {
  0% {transform: scale(1.5) translate(-5em,-24em);}
  100% {transform: scale(1) translate(0,0)}
}

</style>
