<template>
  <div id="app" class="custom-colors custom-fonts flex">
    <div class="picker-container">
      <TimePicker
        :value="timeValue"
        id="timepicker-1"
        @input="changeTime"
        :interval="timeInterval"
        :timeLimits="isAvailiableAll ? availiableTimesAll : availiableTimes"
        :initDateWithClosestInterval="false"
        :allowCustomUserDate="allowCustomUserDate"
        :listHeight="150"
        :placeholder="placeholderText"
        :class="{ 'error-border': dateValue != null && hasError }"
        :isMobile="isMobile"
        :disabled="dateValue == null"
        :headerText="isMobile ? timePickerLabelText : null"
        :format="format"
        inputClass="input-h35"
      />
    </div>
    <div>
      <div>
        <input id="isMobile-id" type="checkbox" v-model="isMobile" />
        <label for="isMobile-id">isMobile: Show mobile view?</label>
      </div>
      <div>
        <input id="isAvailiableAll-id" type="checkbox" v-model="isAvailiableAll" />
        <label for="isAvailiableAll-id">Availiable time: if true - whole day, else - from 7-8, 16:30-17:45</label>
      </div>
      <div>
        <label for="time-interval-id">timeInterval: Time interval in min</label>
        <input id="time-interval-id" type="text" v-model="timeInterval" />
      </div>
      <div>
        <input id="allowCustomUserDate-id" type="checkbox" v-model="allowCustomUserDate" />
        <label for="allowCustomUserDate-id">
          allowCustomUserDate: when true, user can pick any time using keyboard
        </label>
      </div>
      <div>
        <label for="format-id">
          format: Format of the timepicker <a href="https://momentjs.com/docs/#/displaying/format/">Formats</a>
        </label>
        <input id="format-id" type="text" v-model="format" />
      </div>
      <div>
        <label for="placeholderText-id">Placeholder</label>
        <input id="placeholderText-id" type="text" v-model="placeholderText" />
      </div>
      <div>
        <label for="timePickerLabelText-id">dropdown title (in mobile view)</label>
        <input id="timePickerLabelText-id" type="text" v-model="timePickerLabelText" />
      </div>
      <div>
        <input id="show-error-id" type="checkbox" v-model="hasError" />
        <label for="show-error-id">Show error</label>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { Vue, Component } from "vue-property-decorator";
import TimePicker from "@/components/timePicker.vue";
import moment from "moment";

// customize inputs and scroll (for all app)
import "@/styles/_input.scss";
import "@/styles/_scroll.scss";
import "@/styles/_error.scss";
@Component({
  components: { TimePicker },
})
export default class Mrrr extends Vue {
  private isMobile = false;
  private timeValue: Date | null = new Date();
  private allowCustomUserDate = false;
  private timeInterval = 5;

  private isAvailiableAll = true;
  private availiableTimesAll = [moment().startOf("day").toDate(), moment().endOf("day").toDate()];
  private availiableTimes = [
    {
      min: moment().startOf("day").add(7, "hour").toDate(),
      max: moment().startOf("day").add(8, "hour").toDate(),
    },
    {
      min: moment().startOf("day").add(16, "hour").add(30, "minute").toDate(),
      max: moment().startOf("day").add(17, "hour").add(45, "minute").toDate(),
    },
  ];
  private dateValue = new Date();
  private format = "hh:mm a";
  private hasError = false;

  private placeholderText = "Select time";
  private timePickerLabelText = "Pick a time";

  private changeTime(date: Date | null) {
    this.timeValue = date;
  }
}
</script>

<style lang="scss">
html,
body {
  background: rgb(206, 203, 203);
}

.flex {
  display: flex;
}

.picker-container {
  width: 300px;
  margin: 10px 25px;
}

.custom-colors {
  --color-primary: #1c69a8;
  --color-primary-contrast: white;

  --color-list-selection-background: #2080ce;
  --color-list-selection-font: white;

  --color-edit-font: #96aa99;
  --color-edit-background: #fafbfc;
  --color-edit-border: #dfe1e6;

  --color-scroll-background: #f4f5f7;
  --color-scroll-border: #dfe1e6;
}

.custom-fonts {
  --font-family-default: "Roboto", "Arial", "Helvetica", sans-serif;
  --font-default-body-size: 14px;
}

.font-normal {
  font-family: var(--font-family-default);
  font-size: var(--font-default-body-size);
  line-height: 20px;
}
</style>
