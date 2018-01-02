<template>
<div>
    <!-- <p>Errors: {{ errors }}</p> -->
    <div class="title-container">
        <h1>Black Desert Boss Timers</h1>
    </div>
    <br>
    <div class="boss-table">
        <form id="search">
            Search <input name="query" v-model="searchQuery">
            Time zone:
            <select v-model="timezone" name="timezone">
                <option value="America/Vancouver">PST</option>
                <option value="America/Edmonton">MST</option>
                <option value="America/Regina">CST</option>
                <option value="America/Toronto">EST</option>
                <option value="America/Halifax">AST</option>
                <option value="Europe/London">GMT</option>
                <option value="Europe/Luxembourg">CET</option>
                <option value="America/Anchorage">AKST</option>
            </select>
        </form>
        <vue-grid
            :data="gridData"
            :columns="gridColumns"
            :filter-key="searchQuery">
        </vue-grid>
    </div>
</div>
</template>

<script>
import axios from "axios";
import JSSoup from "jssoup";
import moment from "moment-timezone";

import vueGrid from "./Grid.vue";

export default {
  components: {
    vueGrid
  },
  data() {
    return {
      bosses: [
        "Kutum",
        "Dastard Bheg",
        "Dim Tree Spirit",
        "Giant Mudster",
        "Karanda",
        "Kzarka",
        "Red Nose",
        "Nouver"
      ],
      spawnTimes: {},
      timerPage: [],
      nextSpawns: {},
      errors: [],
      gridColumns: [
        "name",
        "lastSpawn",
        "spawnWindowOpen",
        "spawnWindowClose",
        "estSpawn"
      ],
      gridData: [],
      searchQuery: "",
      timezone: "America/Vancouver"
    };
  },
  watch: {
    timezone: function(val) {
      var data = [];
      for (var boss in this.spawnTimes) {
        data.push({
          name: boss,
          lastSpawn: this.spawnTimes[boss].lastSpawn
            .tz(this.timezone)
            .format("MMM Do, HH:mm (ddd)"),
          spawnWindowOpen: this.spawnTimes[boss].spawnWindowOpen
            .tz(this.timezone)
            .format("MMM Do, HH:mm (ddd)"),
          spawnWindowClose: this.spawnTimes[boss].spawnWindowClose
            .tz(this.timezone)
            .format("MMM Do, HH:mm (ddd)"),
          estSpawn: this.spawnTimes[boss].estSpawn
            .tz(this.timezone)
            .format("MMM Do, HH:mm (ddd)")
        });
      }
    //   console.log(data);
      this.gridData = data;
    }
  },
  methods: {
    buildDateObject(givenDate) {
      //This takes a very specific input, the format of which is based on what the boss timer page gives (including the timezone)
      //givenDate should be in format "Fri, 14:00 EST". it will split the values in to a list and remove punctuation.
      //then it will convert the day to a numeral representation of the day of the week
      var currentDate = moment(); //May need to add .tz("America/New_York")  I'm not sure exactly how .tz works yet though.
      var compositeDate = "";
      const weekDays = {
        Sun: 0,
        Mon: 1,
        Tue: 2,
        Wed: 3,
        Thu: 4,
        Fri: 5,
        Sat: 6
      };
      givenDate = givenDate.replace(",", "").split(" ");
      givenDate[0] = weekDays[givenDate[0]];
      //givenDate now looks like this: ['6', '14:00', 'EST']

      if (givenDate[2].includes("EST")) {
        compositeDate = moment.tz(
          `${currentDate.year()}-${currentDate.month() +
            1}-${currentDate.date()} ${givenDate[1]}:00`,
          "America/New_York"
        );
      } else {
        console.log(givenDate);
        console.log("Timezone is Incorrect; give EST timezone, please.");
      }

      //if the givenDate day of the week == the current day of the week then the composite date we built will be accurate so we return it
      //Otherwise, you just need to add one day to the composite date, boss timers never take more than a day.
      if (currentDate.day() === givenDate[0]) {
        return compositeDate;
      } else {
        compositeDate = compositeDate.add(1, "days");
        return compositeDate;
      }
    }
  },
  created() {
    //to get around CORS headers this GET request will go through allorigins.me, rather than directly sending a request to the intended page
    axios
      .get(
        "https://allorigins.me/get?method=raw&url=" +
          encodeURIComponent("http://urzasarchives.com/bdo/wbtbdo/wbtna/") +
          "&callback=?"
      )
      .then(response => {
        //take response HTML and pick out the spawn time tables and save the data
        this.timerPage = response.data;
        var soup = new JSSoup(this.timerPage);
        var tables = soup.findAll("table");
        var bosses = {};

        for (var i = 0; i < tables.length; i++) {
          var tableObject = {};
          var tableItems = tables[i].getText("").split("\n");

          tableObject.name =
            tables[
              i
            ].nextElement.nextElement.nextElement.nextElement.nextElement._text;
          tableObject.lastSpawn = tableItems[3]
            .replace("\t", "")
            .replace("\t", ""); //removing multiple tabs from text
          tableObject.nextSpawn = tableItems[7]
            .replace("\t", "")
            .replace("\t", "");
          tableObject.estSpawn = tableItems[11]
            .replace("\t", "")
            .replace("\t", "");

          bosses[tableObject.name] = tableObject;
        }

        //go through the spawn time data and convert it to useable datetime objects
        for (var key in bosses) {
          this.spawnTimes[key] = {
            lastSpawn: this.buildDateObject(bosses[key].lastSpawn),
            spawnWindowOpen: this.buildDateObject(
              bosses[key].nextSpawn.split(" - ")[0]
            ),
            spawnWindowClose: this.buildDateObject(
              bosses[key].nextSpawn.split(" - ")[1]
            ),
            estSpawn: this.buildDateObject(bosses[key].estSpawn)
          };

          this.gridData.push({
            name: key,
            lastSpawn: this.buildDateObject(bosses[key].lastSpawn).format(
              "MMM Do, HH:mm (ddd)"
            ),
            spawnWindowOpen: this.buildDateObject(
              bosses[key].nextSpawn.split(" - ")[0]
            ).format("MMM Do, HH:mm (ddd)"),
            spawnWindowClose: this.buildDateObject(
              bosses[key].nextSpawn.split(" - ")[1]
            ).format("MMM Do, HH:mm (ddd)"),
            estSpawn: this.buildDateObject(bosses[key].estSpawn).format(
              "MMM Do, HH:mm (ddd)"
            )
          });
        }
        // console.log(this.spawnTimes);
      })
      .catch(e => {
        this.errors.push(e);
      });
  }
};
</script>

<style scoped>
.title-container {
    width: calc(80% + 10px);
    margin: auto;
    display: flex;
    justify-content: center;
    background-color: aliceblue;
    border-radius: 3px;
    /* justify-content: center; */
    color: rgb(63, 67, 78);
}
.boss-table {
    width: 80%;
    margin: 0 auto 0 auto;
    padding: 5px 5px 10px 5px;
    background-color: aliceblue;
    border-radius: 3px;
}
.boss-list {
  margin: auto;
  width: 80%;
  list-style: none;
}
.boss-container {
  min-height: 80px;
  min-width: 500px;
  background-color: grey;
  margin: auto;
}
form {
    padding-bottom: 5px;
}
</style>
