<template>
<body>
  <div>
    <!-- layout prior exercise: prompts user to login -->
    <div class="is-vhcentered has-text-centered pt-6" style="height: 100%">
      <div class="columns is-hcentered mb-5">
        <img class="image is-hcentered" style="width: 100px" src="./assets/iceberg.png" />
      </div>

      <div v-if="!VMAssigned">
        <div class="is-json title">
          Welcome to {{scenarioName}}!
          <div class="subtitle mb-6">A Cyber Range for SOC Analysts.</div>
        </div>
      </div>

      <div v-if=quizActivated>
      <div v-if="!checked" class="subtitle mb-6 is-6">
        Before registration, please take part in our pre-quiz
        <!--strong><a class="subtitle is-4 is-json has-text-link" href="https://quizizz.com/join?gc=07101609" target="_blank"><u>
        pre-quiz</u></a></strong-->
        , then come back to this page.
        <br />Please use your
        <strong>{{naming}} ({{namingPlaceholder}})</strong> to
        register for the quiz.
        <br />

        <div class="buttons is-centered pt-6" v-if="!quizStarted">
          <button
            class="button submit-button is-rounded"
            type="submit"
            value="Submit"
            @click="
                //proceedToQuiz();
                quizStarted = true;
              "
            v-if="!VMsTaken"
          >
            <span>Start Quiz &#10140;</span>
          </button>
        </div>

        <iframe
          v-if="quizStarted"
          :src="urlPreQuiz"
          style="display: block; width: 100%; height: 50vh"
          class="pt-6"
        ></iframe>

        <div v-if="quizStarted" class="mt-5">
          Nothing to see? Click
          <strong>
            <a class="has-text-link" :href="urlPreQuiz" target="_blank">
              <u>here</u>
            </a>
          </strong> and come back to this page when you have finished.
        </div>

        <div class="buttons is-centered pt-6" v-if="quizStarted">
          <button
            class="button submit-button is-rounded"
            type="submit"
            value="Submit"
            @click="checked = true"
            v-if="!VMsTaken"
          >
            <span>Quiz Completed &#10004;</span>
          </button>
        </div>
      </div>
      </div>

      <div v-if="checked && VMAssigned">
        <div class="title pt-6">
          Hello
          <strong class="has-text-primary">{{ this.pseudonym }}</strong>.
        </div>

        <div class="small">(this will be your pseudonym on the cyber range)</div>
        <br />Thank you for registration. We registered your user ID
        <strong
          class="has-text-primary"
        >{{ this.userID }}</strong>
        and set
        up the cyber range for you.
        <div class="buttons is-centered mt-5">
          <button
            class="button submit-button is-rounded mt-5"
            type="submit"
            value="Submit"
            @click="proceedToCR()"
          >
            <span>PROCEED TO CYBER RANGE →</span>
          </button>
        </div>
      </div>

      <br />

      <div stlye="width=100%">
        <div v-if="checked && !VMAssigned" class="subtitle is-5 mb-6">
          <div v-if="quizActivated">
          Thank you for taking part in the pre-quiz!
          <br />Now you can
          <strong>register</strong> for the cyber range
          <br />
          <br />
          <br /> </div>

          <div v-else>
          Please 
          <strong>register</strong> for the cyber range
          <br />
          <br />
          <br /> </div>



          <form @submit.prevent="validateId()" v-if="!VMAssigned">
            <div class="columns is-offset-one-quarter">
              <input
                class="input input-label-long is-size-6 is-centered column"
                :value="'Your '+ naming + ': '"
              />
              <!-- Change Back -->
              <span>
                <input
                  class="input input-short is-size-6 blank-input column"
                  v-model="userID"
                  :placeholder="namingPlaceholder"
                />
              </span>
            </div>

            <div class="has-text-danger" v-if="emptyInput">User ID cannot be empty.</div>
            <div class="has-text-danger" v-if="userExists">User ID was already used.</div>
            <div class="has-text-danger" v-if="VMsTaken">
              No free Cyber Range environments at the moment.
              <br />Please contact one of the trainers.
            </div>

            <br />
            <div class="buttons is-centered">
              <button
                class="button submit-button is-rounded"
                type="submit"
                value="Submit"
                @click="validateId()"
                v-if="!VMsTaken"
              >
                <span>Register</span>
              </button>
            </div>
          </form>
        </div>
      </div>
    </div>
  </div>
</body>
</template>

<script>
import { userDashboard } from "@/firebase"; // TODO rename to userScoreboard
import { VM_db } from "@/firebase";
import settings from "./Settings.js";

export default {
  name: "App",

  data() {
    return {
      gameCompleted: false,
      gameStarted: false,
      userID: null,
      emptyInput: false,
      inputGiven: false,
      userExists: false,
      VMData: null,
      VMAssigned: false,
      VMsTaken: false,
      blockly: null,
      quizActivated: false,
      checked: true, // TODO if quiz activated
      quizStarted: false,
      scenarioName: settings.scenarioName,
      urlPreQuiz: settings.urlPreQuiz,
      naming: settings.naming,
      namingPlaceholder: settings.namingPlaceholder,
    };
  },

  methods: {
    proceedToCR() {
      window.open(this.url, "_self");
    },

    validateId() {
      var message = localStorage.getItem("storedData");
      console.log(message);
      if (this.userID == null) {
        this.userExists = false;
        this.emptyInput = true;
      } else {
        this.emptyInput = false;
        this.inputGiven = true;
        this.userExists = false;
        this.assignVM();
      }
    },

    //randomly retrieve combination of yet unassigned VM/pseudonym
    //create link for user, store userID in localStorage

    assignVM() {
      //create document in user database (cyberrangeDashboard)
      var docRef = userDashboard.doc(String(this.userID)); 
      docRef.get().then((doc) => {
        if (doc.exists) {
          //check if pseudonym was used before
          this.userExists = true;
        } else {
          //register user
          this.getFreeVM();
        }
      });
    },

    async getFreeVM() {
      const snapshot = await VM_db.where("userID", "==", "").limit(1).get();

      this.VMData = snapshot.docs.map((doc) => doc.data());

      if (typeof this.VMData[0] !== "undefined") {
        this.ip = this.VMData[0].ip;
        var blockBool = this.VMData[0].blockly === "true";

        this.blockly = blockBool;

        //const check = await VM_db.doc(this.ip).get();
        //console.log("still empty: ", check.data().userID)

        VM_db.doc(this.ip+this.pseudonym).update({
          userID: this.userID,
        });

        userDashboard.doc(this.userID).set({
          round: this.VMData[0].round,
          level: 0,
          points: 0,
          ip: this.VMData[0].ip,
          pseudonym: this.VMData[0].pseudonym,
          userID: this.userID,
          blockly: this.blockly,
        });

        this.pseudonym = this.VMData[0].pseudonym;
        if (this.blockly == true) {
          this.url =
            "http://" +
            this.VMData[0].ip +
            ":7080?userID=" +
            this.userID +
            "&blockly";
          console.log(this.url);
        } else {
          this.url =
            "http://" + this.VMData[0].ip + ":7080?userID=" + this.userID;
          console.log(this.url);
        }

        //this.url="http://localhost:7080?userID="+this.userID;

        this.VMAssigned = true;
      } else {
        this.VMsTaken = true;
      }
    },
  },

  /*
    getUserPoints() {
      var docRef = userDashboard.doc(String(this.userID));
      docRef
        .get()
        .then((doc) => {
          if (doc.exists) {
            this.round = doc.data().round; //in order to only show the trainees from the same round on the dashboard
            if (doc.data().startTime != null) {
              //get data from user who logged in before
              this.points = doc.data().points;
              this.tasksCompleted = doc.data().level;
              this.startTime = doc.data().startTime;
              if (doc.data().taskTimes != null) {
                this.taskTimes = JSON.parse(doc.data().taskTimes);
              }
            } else {
              //registered player who didn't log in before
              console.log(doc.data().startTime);
              this.tasksCompleted = 0;
              this.startTime = new Date();
              userDashboard.doc(this.userID).update({
                startTime: this.startTime,
              });
              var blanksCompleted = { task2: 0, task3: 0, task4: 0 };
              localStorage.setItem("storedData", JSON.stringify(storedTries));
              localStorage.setItem(
                "blanksCompleted",
                JSON.stringify(blanksCompleted)
              );
            }
          } else {
            // if not only played with preset users
            this.points = 0;
            this.tasksCompleted = 0;
            this.startTime = new Date();
            userDashboard.doc(this.userID).set({
              startTime: this.startTime,
              round: 1,
              level: 0,
              points: 0,
      
              
            });
            console.log(this.getVM()); 
            
          } 
          this.getMarker();
        })
        .catch((error) => {
          console.log("Error getting document:", error);
        });
    },   */

  async getVM() {
    const snapshot = await VM_db.where("userID", "==", "").limit(1).get();
    console.log(snapshot.docs.map((doc) => doc.data()));
    return JSON.stringify(snapshot.docs.map((doc) => doc.data().pseudonym)); //funktioniert so nicht
  },

  async getMarker() {
    const snapshot = await userDashboard
      .where("round", "==", this.round)
      .orderBy("points", "desc")
      .get();
    //const snapshot = await userDashboard.orderBy("points", "desc").get();
    this.dashboard = snapshot.docs.map((doc) => doc.data());
  },

  /*   async uploadPoints() {
      await userDashboard.doc(this.userID).update({
        points: this.points,
        level: this.tasksCompleted,
        startTime: this.startTime,
      });
      this.getMarker();
    },

    async uploadEvaluationData() {
      await userDashboard.doc(this.userID).update({
        taskTimes: JSON.stringify(this.taskTimes),
      });
    },

*/
};
</script>

<style>
@import "./../css/bulma.css";
@import "./../css/bulma-tooltip.css";
</style>
