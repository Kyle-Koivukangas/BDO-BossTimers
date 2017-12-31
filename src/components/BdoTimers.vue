<template>
<div>
    <p>Errors: {{ errors }}</p>
    
    <ul>
        <li></li>
    </ul>
    <p>{{ timerPage }}</p>

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
      jsSoup: "",
      errors: []
    };
  },
  computed: {},
  methods: {
    buildDateObject(givenDate) {
      //This takes a very specific input, the format of which is reliant on what the boss timer page gives (including the timezone)
      //givenDate should be in format "Fri, 14:00 EST", we will split the values in to a list and remove punctuation.
      //then we will convert the day to a numeral representation of the day of the week
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
        // console.log(response);
        this.timerPage = response.data;
        var soup = new JSSoup(this.timerPage);
        var tables = soup.findAll("table");
        var bosses = {};

        for (var i = 0; i < tables.length; i++) {
          var tableObject = {};
          var tableItems = tables[i].getText("").split("\n");

          console.log("table " + i + ":");
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
          console.log(tableObject);

          bosses[tableObject.name] = tableObject;
        }
        console.log(bosses);
        console.log("\nDate Time Objects (last Spawn):");
        for (var key in bosses) {
          let lastSpawnTemp = this.buildDateObject(bosses[key].lastSpawn);
          let spawnWindowOpenTemp = this.buildDateObject(
            bosses[key].nextSpawn.split(" - ")[0]
          );
          let spawnWindowCloseTemp = this.buildDateObject(
            bosses[key].nextSpawn.split(" - ")[1]
          );
          let estSpawnTemp = this.buildDateObject(bosses[key].estSpawn);

          console.log(`${key}\nLast Spawn: ${bosses[key].lastSpawn}`);
          console.log(lastSpawnTemp.format());
          console.log(`\nSpawn Window: ${bosses[key].nextSpawn}`);
          console.log(
            `${spawnWindowOpenTemp.format()} - ${spawnWindowCloseTemp.format()}`
          );
          console.log(`\nEstimated Spawn: ${bosses[key].estSpawn}`);
          console.log(estSpawnTemp.format());
          console.log("\n\n");
          this.spawnTimes[key] = {
              lastSpawn: this.buildDateObject(bosses[key].lastSpawn),
              spawnWindowOpen: this.buildDateObject(bosses[key].nextSpawn.split(" - ")[0]),
              spawnWindowClose:  this.buildDateObject(bosses[key].nextSpawn.split(" - ")[1]),
              estSpawn: this.buildDateObject(bosses[key].estSpawn)
          }
        }
        console.log(this.spawnTimes)

        //Need to figure out how the date time stuff will work...
        //need to convert string date ("Fri, 21:45 EST") to date time object with timezone functionality
        // (function convertToDatetime() {
        //   console.log("converting Datetimes:\n");
        //   var currentDate = new Date();
        //   const weekDays = {
        //     Sun: 0,
        //     Mon: 1,
        //     Tue: 2,
        //     Wed: 3,
        //     Thu: 4,
        //     Fri: 5,
        //     Sat: 6
        //   };
        //   for (var key in bosses) {
        //     console.log(key + ":");
        //     var lastSpawn = bosses[key].lastSpawn.replace(",", "").split(" ");
        //     lastSpawn[0] = weekDays[lastSpawn[0]];

        //     console.log("Last Spawn:");
        //     console.log(lastSpawn);
        //     if (currentDate.getDay() === lastSpawn[0]) {
        //       console.log("\nDays are equal:\n\nComposite date: ");
        //       var compositeDate = moment.tz(
        //         `${currentDate.getFullYear()}-${currentDate.getMonth() +
        //           1}-${currentDate.getDate()} ${lastSpawn[1]}:00`,
        //         "America/New_York"
        //       );
        //       console.log(compositeDate.format());
        //       console.log(`CompositeDate with NewYorkTZ:`);
        //       console.log(compositeDate.tz("America/New_York").format());

        //       var LA_TZ = compositeDate.clone().tz("America/Los_Angeles");
        //       console.log("\nCompositeDate with LA TZ:");
        //       console.log(LA_TZ.format());
        //       console.log("\nAdd one day:");
        //       console.log(LA_TZ.add("days", 1).format());
        //     } else {
        //     }

        //     console.log("currentDate: " + currentDate);
        //     console.log(currentDate.getDay());

        //     // var lastSpawnDate = moment(
        //     //     currentDate.getYear() + '-' +
        //     //     currentDate.getMonth() + '-' +
        //     //     currentDate.
        //     // )
        //     // console.log(lastSpawnDate)
        //     console.log(bosses[key].lastSpawn);
        //     console.log("\n");
        //     // bosses[key].lastSpawn =
        //   }
        // })();

        soup.findAll("");
      })
      .catch(e => {
        this.errors.push(e);
      });
  }
};
</script>

<style scoped>

</style>
