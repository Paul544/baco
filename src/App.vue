<template>
  <b-container id="app" fluid>
    <br />
    <b-button-toolbar key-nav aria-label="Toolbar with button groups">
      <b-button-group class="mx-2">
        <b-button v-on:click="NewTeam">New Team</b-button>
      </b-button-group>
    </b-button-toolbar>
    <br />
    <div v-for="team in Teams" v-bind:key="team.TeamNo">
      <b-card :style="team.Style">
        <b-card-text>
          <b-row>
            <b-col cols="1">
              <span
                ><strong>{{ team.TeamNo }}</strong></span
              >
            </b-col>
            <b-col>
              <b-form-select
                v-if="!team.Started"
                v-model="team.Type"
                :options="Options"
                @change="RefreshData($event, team)"
              ></b-form-select>
              <b-button
                id="start"
                variant="success"
                @click="Start($event, team)"
                v-if="!team.Started"
                >Start</b-button
              >
              <b-button
                id="stop"
                variant="danger"
                @click="Stop($event, team)"
                v-if="team.Started"
                >Stop</b-button
              >
            </b-col>
          </b-row>
          <hr />
          <b-row v-for="ba in team.BA" v-bind:key="ba.Id" style="height: 65px">
            <b-col>
              <b-form-input
                v-model="ba.Name"
                :id="ba.Name"
                placeholder="Enter name"
                v-if="!team.Started"
              ></b-form-input>
              <span v-if="team.Started">{{ ba.Name }}</span>
            </b-col>
            <b-col>
              <b-form-spinbutton
                id="sb-inline"
                v-model="ba.CurrentPressure"
                inline
                :min="0"
                :max="ba.Max"
                step="10"
                @change="UpdateTime($event, team)"
              ></b-form-spinbutton>
            </b-col>
            <hr />
            <br />
          </b-row>
          <b-row>
            <b-col>
              <b-form-timepicker
                now-button
                reset-button
                locale="en"
                v-model="team.TimeSlot"
                @input="test($event, team)"
                :hour12="false"
                v-if="!team.Started"
              ></b-form-timepicker>
              <h2>
                <span v-if="team.Started">{{
                  team.Time | moment("HH:mm")
                }}</span>
              </h2>
            </b-col>
            <b-col>
              <h2>
                {{ team.EndTime | moment("HH:mm") }} /
                {{ team.RemainingMinutes }}
              </h2>
            </b-col>
          </b-row>
        </b-card-text>
      </b-card>
      <br />
    </div>
    
  </b-container>
</template>

<script>
var teamNo = 0;
var baNo = 1;
var audio = new Audio(require('./alarm.mp3'))
import moment from "moment";

export default {
  name: "App",
  components: {},
  data: function () {
    return {
      Teams: [],
      Options: [
        { value: 0, text: "9L 300", min: 200, max: 300, bar: 300, lit: 9000 },
        { value: 1, text: "9L 207", min: 160, max: 200, bar: 200, lit: 9000 },
        { value: 2, text: "6.8L 300", min: 200, max: 300, bar: 300, lit: 6800 },
      ],
    };
  },
  mounted() {
    moment.locale("en");

    setInterval(this.Update, 1000);
  },

  methods: {
    test(e, team) {
      var dateOnly = moment().format("yyyy-MM-DD");

      var currentDate = moment(dateOnly + "T" + team.TimeSlot);

      team.Time = currentDate;
      team.TimeSlot = team.Time.format("HH:mm:ss");
      this.CalcEndTime(team);
    },
    Start(ev, team) {
      team.Started = true;
      team.Style = "background-color:darkgreen;color:white";
      
    },
    Stop(ev, team) {
      team.Started = false;
      team.Style = "background-color:darkgrey;color:white";
      this.RefreshData(ev, team);
      audio.pause();
      audio.currentTime=0;
      team.HasAlarm = false;
    },
    RefreshData(e, team) {
      this.SetMinBarForEachBa(team);
      this.CalcEndTime(team);
    },
    SetMinBarForEachBa(team) {
      var option = this.Options[team.Type];

      for (var baIndex = 0; baIndex <= team.BA.length - 1; baIndex++) {
        let ba = team.BA[baIndex];
        ba.Min = 0;
        ba.Max = option.max;
        ba.CurrentPressure = ba.Max;
      }
    },
    CalcEndTime(team, bar) {
      var opt = this.Options[team.Type];
      if (bar == undefined) {
        bar = opt.bar;
      }

      var minutes = this.GetWorkingTime(bar, opt);
      team.RemainingMinutes = minutes;

      var dateOnly = moment().format("yyyy-MM-DD");

      var currentDate = moment(dateOnly + "T" + team.TimeSlot);
      team.EndTime = moment(currentDate).add(minutes, "minutes") + 1;
    },
    Update() {
      for (var teamIndex = 0; teamIndex <= this.Teams.length - 1; teamIndex++) {
        var team = this.Teams[teamIndex];
        if (!team.Started) {
          team.Time = moment();
          this.TimeSlot = team.Time.format("Hh:mm:ss");
          this.UpdateTime(null, team);
        } else {
          this.CalcEndTime(team, this.GetMinBar(team));
          var minutes = moment(team.EndTime).diff(this.Time, "minutes");
          team.RemainingMinutes = minutes + 1;
          if (team.RemainingMinutes > 15) {
             team.HasAlarm=false
             audio.pause();
             audio.currentTime=0;
            team.Style = "background-color:green;color:white";
          } else if (team.RemainingMinutes > 10) {
            team.HasAlarm=false
            audio.pause();
             audio.currentTime=0;
            team.Style = "background-color:orange;color:white";
          } else {
            if (!team.HasAlarm){
             team.HasAlarm=true;
             audio.currentTime=0;
             this.Play(); 
            }
            team.Style = "background-color:red;color:white";
          }
        }
      }
    },

    GetWorkingTime(bar, option) {
      return Math.round((bar * option.lit) / 1000 / 40 - 0.4, 0);
    },

    GetMinBar(team) {
      var minValue = 0;
      for (var ba = 0; ba <= team.BA.length - 1; ba++) {
        var baItem = team.BA[ba];

        if (minValue == 0 || baItem.CurrentPressure < minValue) {
          minValue = baItem.CurrentPressure;
        }
      }

      return minValue;
    },

  Play() {   
      audio.currentTime=0;
      audio.play();
    } ,

    UpdateTime($event, team) {
      
       this.CalcEndTime(team, this.GetMinBar(team));
          var minutes = moment(team.EndTime).diff(this.Time, "minutes");
          team.RemainingMinutes = minutes + 1;
    },

    NewTeam: function () {
      teamNo++;
      baNo += 2;

      var team = {
        TeamNo: teamNo,
        Type: 0,
        Time: "",
        TimeSlot: "",
        EndTime: "",
        EndSlot: "",
        Remaining: 100,
        Minutes: 0,
        RemainingMinutes: 0,
        Started: false,
        HasAlarm: false,
        Style: "background-color:darkgrey;color:white",
        BA: [
          {
            Id: baNo - 1,
            Name: "",
            CurrentPressure: 300,
            Min: 200,
            Max: 300,
          },
          {
            Id: baNo,
            Name: "",
            CurrentPressure: 300,
            Min: 200,
            Max: 300,
          },
        ],
      };
      team.Time = moment();
      team.TimeSlot = team.Time.format("HH:mm:ss");
      team.EndTime = moment();

      this.Teams.push(team);
      this.RefreshData(null, team);
    },
  },
};
</script>

<style scoped>
#start,
#stop {
  margin-left: 20px;
}
</style>