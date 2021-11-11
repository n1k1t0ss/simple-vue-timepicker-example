<template>
  <div class="timepicker input-default" :class="{ 'bp-mobile': isMobile }">
    <input
      ref="input"
      v-bind="$attrs"
      type="text"
      :value="readableValue"
      class="timepicker-input font-normal"
      :class="inputClass"
      :placeholder="placeholder"
      :disabled="disabled"
      @input="onInput($event.target.value)"
      @focus.prevent="onFocus"
      @change="onInputEnd($event.target.value)"
      @click="onClick"
      @blur.prevent="onBlur"
      @keydown.down.prevent="pointerForward()"
      @keydown.up.prevent="pointerBackward()"
      @keypress.enter.prevent.stop.self="onEnter"
      :style="{ height: optionHeight + 'px' }"
      :readonly="isMobile ? 'readonly' : null"
    />
    <span class="list-opener" @click="onOpenerClick"></span>
    <transition name="timepicker">
      <div
        ref="list"
        class="dropdown-content-wraper custom-scroll font-normal"
        :class="{ 'scroll--small': !isMobile }"
        v-show="isOpen"
        tabindex="-1"
        @mousedown.prevent
        :style="{ maxHeight: isMobile ? '100vh' : listHeight + 'px' }"
      >
        <div
          v-if="headerText"
          class="timepicker__option timepicker__header"
          :style="{
            height: isMobile ? optionMobileHeight + 'px' : optionHeight + 'px',
          }"
        >
          {{ headerText }}
        </div>
        <ul class="dropdown-content">
          <li
            class="timepicker__option"
            v-for="(timeoption, index) in timeOptions"
            :key="`opt-${timeoption}-${index}`"
            :value="timeoption"
            @click.stop="onDropdownClick(timeoption)"
            :class="optionHighlight(index, timeoption)"
            @mouseenter.self="pointerSet(index)"
            :style="{
              height: isMobile ? optionMobileHeight + 'px' : optionHeight + 'px',
            }"
          >
            <span class="timepicker__option__span">{{ timeoption }}</span>
          </li>
        </ul>
      </div>
    </transition>
  </div>
</template>

<script lang="ts">
import { Vue, Component, Prop, Watch, Emit } from "vue-property-decorator";
import moment from "moment";

@Component({ inheritAttrs: false })
export default class TimePicker extends Vue {
  // refs
  $refs!: {
    input: HTMLFormElement;
    list: HTMLFormElement;
  };
  // props
  // Input only, use dateless valueTime
  @Prop({ default: null })
  private readonly value!: Date;

  @Prop({ default: 15 })
  private readonly interval!: number;

  @Prop({ default: null, required: true })
  private readonly timeLimits!: { min: Date; max: Date }[];

  // can user add date that doesn't much interval
  @Prop({ default: true })
  private readonly allowCustomUserDate!: boolean;

  // if true, initial time will be changed to closest interval date
  @Prop({ default: false })
  private readonly initDateWithClosestInterval!: boolean;

  @Prop({ default: "hh:mm a" })
  private readonly format!: string;

  @Prop()
  private readonly headerText!: string;

  @Prop({ default: 40 })
  private readonly optionHeight!: number;

  @Prop({ default: 65 })
  private readonly optionMobileHeight!: number;

  @Prop({ default: 200 })
  private readonly listHeight!: number;

  @Prop()
  private readonly placeholder!: string;

  @Prop()
  private readonly disabled!: string;

  @Prop()
  private readonly isMobile!: boolean;

  @Prop({ default: "input-h40" })
  private readonly inputClass!: string;

  // variables
  private isOpen = false;
  private search: string | null = null;
  private selectedPosition = 0;
  private customUserDate: Date | null = null;

  // computed
  get timeOptionsRaw(): Date[] {
    const result: Date[] = [];
    for (let i = 0; i < this.timeLimits.length; i++) {
      const min = this.timeLimits[i].min;
      const max = this.timeLimits[i].max;
      const minTime = min ? moment(min) : moment().startOf("day");
      let current = moment().startOf("day");
      const maxTime = max ? moment(max) : moment().endOf("day");
      while (current.isSameOrBefore(maxTime)) {
        const nextDate = current.clone().add(this.interval, "minute");
        if (current.isSameOrAfter(minTime)) {
          result.push(current.toDate());
        }

        // add custom user's date
        if (this.allowCustomUserDate && this.customUserDate) {
          const customDate = moment(this.customUserDate);
          if (
            customDate.isAfter(current) &&
            customDate.isBefore(nextDate) &&
            customDate.isSameOrAfter(minTime) &&
            customDate.isSameOrBefore(maxTime)
          ) {
            result.push(customDate.toDate());
          }
        }

        current = nextDate;
      }
    }
    return result;
  }

  get timeOptions(): string[] {
    return this.timeOptionsRaw.map((o) => this.formatTo(o));
  }

  get readableValue(): string {
    if (this.search !== null) {
      return this.search;
    }

    if (!this.valueTime) {
      return "";
    }
    return this.valueTime ? this.formatTo(moment(this.valueTime).toDate()) : this.valueTime;
  }

  get pointerPosition(): number {
    return this.selectedPosition * (this.isMobile ? this.optionMobileHeight : this.optionHeight);
  }

  get visibleElementsCount() {
    return this.listHeight / (this.isMobile ? this.optionMobileHeight : this.optionHeight);
  }

  get pointerPositionEnd() {
    return (
      this.pointerPosition -
      (this.visibleElementsCount - 1) * (this.isMobile ? this.optionMobileHeight : this.optionHeight)
    );
  }

  get pointerPositionMiddle() {
    return (
      this.pointerPosition -
      ((this.visibleElementsCount - 1) * (this.isMobile ? this.optionMobileHeight : this.optionHeight)) / 2
    );
  }

  private get valueTime(): Date | null {
    return this.CreateDate(this.value);
  }

  // methods
  private created() {
    this.onValueChanged();
    if (this.initDateWithClosestInterval) {
      const value = moment(this.valueTime!);
      const result = this.findClosestInterval(value.toDate()).date;
      if (result && !value.isSame(result)) {
        this.saveDate(result);
      }
    }
  }

  private convertFrom(value: string): moment.Moment {
    return moment(value, this.format);
  }

  private formatTo(date: Date): string {
    return moment(date).format(this.format);
  }

  private onDropdownClick(option: string) {
    this.onBlur();
    this.saveDate(this.convertFrom(option).toDate());
  }

  @Watch("value")
  private onValueChanged(): void {
    if (this.value && this.timeLimits && this.timeLimits.length > 0) {
      if (this.isValueOutOfLimits(this.value)) {
        const result = this.findClosestInterval(this.valueTime!).date;
        if (!this.isValueOutOfLimits(result)) {
          this.saveDate(result);
        }
      }
    }
  }

  private isValueOutOfLimits(value: Date): boolean {
    const val = moment(this.CreateDate(value, value)!);
    const interval = this.timeLimits.find(
      (o) => val.isSameOrAfter(this.CreateDate(o.min, value)!) && val.isSameOrBefore(this.CreateDate(o.max, value)!)
    );
    return !interval;
  }

  @Watch("timeLimits")
  private checkTimeLimits(): void {
    this.onValueChanged();
  }

  private findClosestInterval(date: Date): { index: number; date: Date } {
    let maxIndex = this.timeOptionsRaw.findIndex((o) => moment(date).isSameOrBefore(o));

    let minIndex = 0;
    if (maxIndex == -1) {
      maxIndex = this.timeOptionsRaw.length - 1;
    } else {
      minIndex = maxIndex > 0 ? maxIndex - 1 : maxIndex;
    }

    if (minIndex == maxIndex) {
      return { index: minIndex, date: this.timeOptionsRaw[minIndex] };
    }

    const minDiff = Math.abs(moment(date).diff(this.timeOptionsRaw[minIndex]));
    const maxDiff = Math.abs(moment(date).diff(this.timeOptionsRaw[maxIndex]));

    const index = minDiff < maxDiff ? minIndex : maxIndex;
    return { index: index, date: this.timeOptionsRaw[index] };
  }

  private onInput(value: string): void {
    this.isOpen = true;
    this.search = value;
    this.customUserDate = null;

    this.scrollListTo(value);

    const date = this.convertFrom(value);
    if (date.isValid()) {
      const converted = this.formatTo(date.toDate());

      const newIndex = this.timeOptions.findIndex((o) => o == converted);

      if (newIndex !== -1) {
        this.selectedPosition = newIndex;
      } else {
        if (this.allowCustomUserDate) {
          this.customUserDate = date.clone().toDate();
          this.selectedPosition = this.timeOptions.findIndex((o) => o == converted);
        } else {
          this.selectedPosition = this.findClosestInterval(date.toDate()).index;
        }
      }
      this.$refs.list.scrollTop = this.pointerPositionMiddle;
    }
  }

  private scrollListTo(value: string) {
    const trimVal = value.trim();
    if (trimVal.length > 1) {
      for (let index = 0; index < this.timeOptions.length; ++index) {
        if (this.timeOptions[index].indexOf(trimVal) !== -1) {
          this.selectedPosition = index;
          this.$refs.list.scrollTop = this.pointerPositionMiddle;
          return;
        }
      }
    }
  }

  private onInputEnd(value: string): void {
    let date = this.convertFrom(value);
    if (!date.isValid()) {
      // restore date
      date = moment(this.valueTime!);
      this.saveDate(date.toDate());
      return;
    }

    if (!this.allowCustomUserDate) {
      const result = this.findClosestInterval(date.toDate()).date;
      this.saveDate(result);
    } else {
      this.saveDate(date.toDate());
    }
  }

  // create new date from time and date. if date is null - use today
  private CreateDate(time: Date | null, date: Date | null = null): Date | null {
    if (!time) {
      return null;
    }

    const mDate = date ? moment(date) : moment();
    return mDate.minute(time.getMinutes()).hour(time.getHours()).second(0).millisecond(0).toDate();
  }

  private onEnter(): void {
    if (this.isOpen && this.selectedPosition !== -1 && this.selectedPosition < this.timeOptions.length - 1) {
      this.onDropdownClick(this.timeOptions[this.selectedPosition]);
    }
    if (this.search !== null) {
      this.isOpen = false;
      this.onInputEnd(this.search);
    }
  }

  private onFocus(event: any): void {
    event.target.select();
  }

  private onOpenerClick(): void {
    if (!this.disabled) {
      this.$refs.input.select();
      this.onClick();
    }
  }

  private async onClick() {
    this.isOpen = true;
    await this.$nextTick();
    this.scrollListTo(this.readableValue);
  }

  private onBlur(): void {
    this.$emit("blur");
    this.isOpen = false;
  }

  private pointerForward(): void {
    this.isOpen = true;
    if (this.selectedPosition < this.timeOptions.length - 1) {
      this.selectedPosition++;
      if (this.$refs.list.scrollTop <= this.pointerPositionEnd) {
        this.$refs.list.scrollTop = this.pointerPositionEnd;
      }
    }
  }

  private pointerBackward(): void {
    this.isOpen = true;
    if (this.selectedPosition > 0) {
      this.selectedPosition--;
      if (this.$refs.list.scrollTop >= this.pointerPosition) {
        this.$refs.list.scrollTop = this.pointerPosition;
      }
    }
  }

  pointerSet(index: number): void {
    this.selectedPosition = index;
  }

  private optionHighlight(index: number, option: string) {
    return {
      "timepicker__option--highlight": index === this.selectedPosition,
      "timepicker__option--selected": option === this.readableValue,
    };
  }

  // @Emit("input", )
  private saveDate(date: Date | null): void {
    this.search = null;
    this.$emit("input", this.CreateDate(date, this.value));
  }
}
</script>

<style lang="scss" scoped>
.timepicker {
  position: relative;
}

.timepicker-input:disabled {
  background-color: #eff0f3;
}

.dropdown-content-wraper {
  position: absolute;
  display: block;
  background: #fff;
  width: 100%;
  max-height: 240px;
  overflow: hidden auto;
  border: 1px solid #e8e8e8;
  border-top: none;
  border-bottom-left-radius: 5px;
  border-bottom-right-radius: 5px;
  z-index: 50;
  -webkit-overflow-scrolling: touch;
  overflow: auto;
}

.dropdown-content {
  list-style: none;
  display: inline-block;
  padding: 0;
  margin: 0;
  min-width: 100%;
  vertical-align: top;
  height: 100%;
  display: inline-block;
}

.dropdown-content::webkit-scrollbar {
  display: none;
}

.timepicker__option {
  display: block;
  padding: 0 12px;
  min-height: 15px;
  line-height: 14px;
  text-decoration: none;
  text-transform: none;
  position: relative;
  cursor: pointer;
  white-space: nowrap;

  display: flex;
  align-items: center;
}

.bp-mobile .timepicker__option {
  border-width: 0 0 1px 0;
  border-style: solid;
  border-color: #e1edf3;
  padding: 0 24pt;
}

.timepicker__option:after {
  top: 0;
  right: 0;
  position: absolute;
  line-height: 40px;
  padding-right: 12px;
  padding-left: 20px;
}

.timepicker__option__span {
  color: #616d82;
}

.timepicker__option--highlight {
  outline: none;
  background-color: var(--color-list-selection-background);
  .timepicker__option__span {
    color: var(--color-list-selection-font);
  }
}

.timepicker__option--highlight:after {
  content: attr(data-select);
  background-color: var(--color-list-selection-background);
  color: var(--color-list-selection-font);
}

.timepicker__option.timepicker__option--selected {
  background-color: var(--color-primary);

  .timepicker__option__span {
    color: var(--color-primary-contrast);
  }
}

.timepicker__option--selected:after {
  content: attr(data-selected);
  color: silver;
}

.bp-mobile .timepicker__option .timepicker__option__span {
  font-size: 14pt;
  letter-spacing: 2pt;
}

.dropdown__element {
  display: block;
}

.dropdown :hover {
  background-color: #ddd;
}

.timepicker-enter-active,
.timepicker-leave-active {
  transition: all 0.3s;
}

.timepicker-enter {
  transform-origin: top;
  transform: scaleY(0);
}

.timepicker-leave-to {
  opacity: 0;
}

.bp-mobile .dropdown-content-wraper {
  max-height: 100vh;
  top: 0px;
  right: 0px;
  position: fixed;
  max-height: 100vh;
  height: 100%;
}

.list-opener {
  position: absolute;
  text-align: center;
  bottom: 0;
  right: 0;
  top: 6px;
  width: 26px;
  color: #dfe1e6;

  &:after {
    content: "";
    border: solid #9697aa;
    border-width: 0 1.5px 1.5px 0;
    transform: rotate(45deg);
    -webkit-transform: rotate(45deg);
    display: inline-block;
    padding: 3px;
  }
}
.timepicker__header {
  position: sticky;
  top: 0;
  background: white;
  height: 100px;
  width: 100%;
  z-index: 1;

  color: #99a2af;
  font-size: 12px;
}

.bp-mobile .timepicker__header {
  font-size: 12pt;
}
</style>
