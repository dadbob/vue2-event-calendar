<template>
  <div :class="[`${prefixCls}`, `is-${mode}`]">
    <calendar-header
      :mode="mode"
      :prefix-cls="prefixCls"
      :render-header="renderHeader"
      :header-left="$slots['header-left']"
      :header-right="$slots['header-right']"
      :current-date="formatedDay"
      @prev="prev"
      @next="next" />

    <div :class="`${prefixCls}-week`">
      <div v-for="item in titleArray"
        :key="item"
        :class="`${prefixCls}-week__item`">
        {{ item }}
      </div>
    </div>

    <div :class="`${prefixCls}-body`">
      <div v-for="(row, index) in (monthData.length / titleArray.length)"
        :key="index"
        :class="`${prefixCls}-body-row`">
        <template v-for="i in 7">
          <div :class="`${prefixCls}-day-item`"
            v-if="monthData[(i - 1) + index * 7]"
            :key="i">
            <slot :date="monthData[(i - 1) + index * 7]">
              <span>{{ monthData[(i - 1) + index * 7].date.date }}</span>
            </slot>
          </div>
        </template>
       </div>
    </div>
  </div>
</template>

<script>
import dayjs from 'dayjs';
import getMonthViewStartDay from './date-func';
import CalendarHeader from './header';

export default {
  name: 'VueCalendar',
  components: {
    CalendarHeader
  },
  props: {
    prefixCls: {
      type: String,
      default: 'calendar'
    },
    startDate: [Number, String, Date],
    dateData: {
      type: [Object, Array],
      default: () => []
    },
    matchKey: {
      type: String,
      default: 'date'
    },
    locale: {
      type: String,
      default: 'en'
    },
    firstDay: {
      type: Number,
      default: 0
    },
    mode: {
      type: String,
      default: 'month',
      validator: val => val === 'month' || val === 'week'
    },
    weekDateShort: {
      type: Array,
      validator: val => val.length === 7
    },
    renderHeader: Function,
    weekLocaleData: Array
  },
  data() {
    return {
      today: this.currentDay,
      currentDay: null,
      localeData: {
        'zh-cn': '周日_周一_周二_周三_周四_周五_周六'.split('_'),
        en: 'Sun_Mon_Tue_Wed_Thu_Fri_Sat'.split('_')
      }
    };
  },
  computed: {
    formatedDay() {
      return dayjs(new Date(this.currentDay));
    },

    titleArray() {
      const arr = this.weekDateShort || this.weekLocaleData || this.localeData[this.locale];
      let i = this.firstDay - 1;

      return arr.map(() => {
        i += 1;
        if (i >= 7) { i = 0; }

        return arr[i];
      });
    },

    monthData() {
      const {
        dateData,
        formatedDay,
        firstDay,
        mode,
        matchKey
      } = this;

      if (!formatedDay) { return []; }

      // start date of view, and it will be
      let startDate = getMonthViewStartDay(
        formatedDay,
        firstDay,
        mode,
      );
      const monthData = [];
      let row = 6;

      // change row lenght when mode changing
      if (this.mode === 'week') { row = 1; }

      // loop view item and get date data
      for (let day = 0; day < 7 * row; day += 1) {
        let data = [];

        // TODO:
        // opmize ALG

        /* eslint no-loop-func: 0 */
        // get data if date matched
        if (dateData instanceof Array) {
          data = dateData.filter((item) => {
            const date = item[matchKey].replace('-', '/');
            return startDate.isSame(
              dayjs(new Date(date)),
            );
          });
        } else {
          Object.keys(dateData).forEach((key) => {
            const date = key.replace('-', '/');
            if (startDate.isSame(dayjs(new Date(date)))) {
              data.push(dateData[key]);
            }
          });
        }

        // get date info
        monthData.push({
          ...this.getItemStatus(startDate),
          data: data || {},
          date: this.getDate(startDate)
        });

        // increase date
        startDate = startDate.add(1, 'day');
      }

      return monthData;
    }
  },
  watch: {
    startDate: {
      immediate: true,
      handler(val) {
        this.currentDay = val ? new Date(val) : new Date();

        if (!this.today) {
          this.today = this.currentDay;
        }
      }
    },
    currentDay: {
      immediate: true,
      handler: 'onMonthChange'
    },
    mode: 'onMonthChange'
  },
  methods: {
    getItemStatus(date) {
      const tempDate = dayjs(date);
      const {
        formatedDay
      } = this;

      const isCurMonth = tempDate.month() === formatedDay.month();

      const isPrevMonth = !isCurMonth && tempDate.isBefore(this.formatedDay, 'month');
      const isNextMonth = !isCurMonth && tempDate.isAfter(this.formatedDay, 'month');

      const isPrevLastDay = isPrevMonth ? tempDate.isSame(tempDate.endOf('month')) : false;
      const isNextFirstDay = isNextMonth ? tempDate.isSame(tempDate.startOf('month')) : false;

      return {
        isPrevMonth,
        isPrevLastDay,
        isNextMonth,
        isNextFirstDay,
        // isToday: date.isSame(dayjs(this.today), 'day'),
        isToday: date.format('YYYY-MM-DD') === dayjs(this.today).format('YYYY-MM-DD'),
        isCurMonth
      };
    },

    getDate(date) {
      return {
        year: date.year(),
        month: date.month() + 1,
        date: date.date(),
        day: date.day(),
        full: date.format('YYYY-MM-DD')
      };
    },

    onMonthChange() {
      this.$emit('onMonthChange', {
        startDay: this.monthData[0].date,
        endDay: this.monthData[this.monthData.length - 1].date,
        now: this.getDate(this.formatedDay)
      });
    },

    changeDate(date) {
      if (typeof date !== 'string' && Object.prototype.toString.call(date) !== '[object Date]') {
        /* tslint:disable: no-console */
        console.error('invalied date!');
        return;
      }

      this.currentDay = date;
    },

    prev() {
      const {
        formatedDay,
        mode,
        monthData
      } = this;

      this.currentDay = formatedDay
        .subtract(1, mode)
        .startOf(mode)
        .format('YYYY-MM-DD');

      this.$emit('prev', {
        startDay: monthData[0].date,
        endDay: monthData[monthData.length - 1].date,
        now: this.getDate(formatedDay)
      });
    },

    next() {
      const {
        formatedDay,
        mode,
        monthData
      } = this;

      this.currentDay = formatedDay
        .add(1, mode)
        .startOf(mode)
        .format('YYYY-MM-DD');

      this.$emit('next', {
        startDay: monthData[0].date,
        endDay: monthData[monthData.length - 1].date,
        now: this.getDate(formatedDay)
      });
    }
  }
};
</script>

<style lang="less">
@import "./style/calendar";
</style>