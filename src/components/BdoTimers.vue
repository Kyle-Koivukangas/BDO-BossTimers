<template>
<div>
    <p>Errors: {{ errors }}</p>
    
    <ul class="boss-list">
        <li v-for="boss in bosses" :key="boss">
            <div class="boss-container">
                <h1>{{ boss }}</h1>
            </div>
        </li>
    </ul>
    <p>{{ spawnTimes }}</p>

</div>
</template>

<script>
import axios from "axios";
import JSSoup from "jssoup";
import moment from "moment-timezone";

export default {
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
      errors: []
    };
  },
  computed: {},
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
        }
        console.log(this.spawnTimes);
      })
      .catch(e => {
        this.errors.push(e);
      });
  }
};
</script>

<style scoped>
.boss-list {
    margin: auto;
    width: 80%;
}
.boss-container {
    min-height: 80px;
    min-width: 500px;
    background-color: grey;
    margin: auto;
}
</style>
