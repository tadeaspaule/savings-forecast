<template>
  <q-page class="flex flex-center" style="flex-direction:column;">
    <div style="margin-top:-150px;margin-bottom:100px;">
      This site is for calculating how much and how long you need to save to reach your financial independance goals
    </div>
    <div style="display:flex;flex-direction:row;justify-content:center;">
      <div class="slider-sidebar">
        <div class="slider">
          Monthly saving: {{monthlySaving}}€
          <q-slider :min="0" :max="10000" v-model="monthlySaving" flat dense />
        </div>
        <div class="slider">
          Start amount: {{startAmount}}€
          <q-slider :min="0" :max="10000" v-model="startAmount" flat dense />
        </div>
        <div class="slider">
          <div>
            Savings mode:
            <HelpTooltip text="Target: Save until reaching a specific number Expenses: Save until returns >= expenses" />
            <q-radio v-model="targetOrExpenses" val="target" label="Target" />
            <q-radio v-model="targetOrExpenses" val="expenses" label="Expenses" />
          </div>
          {{targetOrExpenses == 'target' ? `Target: ${savingTarget}€` : `Monthly expenses: ${monthlyExpenses}€ (target will be ${Math.round(target)}€)`}}
          <q-slider :min="10000" :max="1000000" v-model="savingTarget" flat dense v-if="targetOrExpenses == 'target'"/>
          <q-slider :min="300" :max="10000" v-model="monthlyExpenses" flat dense v-if="targetOrExpenses == 'expenses'"/>
        </div>
        <div class="slider">
          <span>
            Annual returns: <HelpTooltip text="How much do you expect your investments to grow each year?" />
            {{yearlyReturn}}%
          </span>
          <q-slider :min="0" :max="50" :step="0.1" v-model="yearlyReturn" flat dense />
        </div>
        <div class="slider">
          <div>
            Returns mode:
            <q-radio v-model="returnsMode" val="yearly" label="Annual" />
            <q-radio v-model="returnsMode" val="quarterly" label="Quarterly" />
          </div>
        </div>
        <div class="slider">
          <span>
            Increase monthly saving every {{increaseMonthlyEvery}} months
            <HelpTooltip text="Leave at 0 if you never want to increase the amount" />
          </span>
          <q-slider :min="0" :max="50" v-model="increaseMonthlyEvery" flat dense />
        </div>
        <div class="slider">
          <span>
            Increase monthly saving by {{increaseMonthlyBy}}
            <HelpTooltip text="Related to the slider above" />
          </span>
          <q-slider :min="0" :max="1000" v-model="increaseMonthlyBy" flat dense :disable="increaseMonthlyEvery == 0"/>
        </div>
      </div>
      <div class="saving-forecast">
        Months until target reached: {{statusEveryMonth.length}} ({{`${Math.ceil(statusEveryMonth.length / 12)} years`}})
      </div>
    </div>
  </q-page>
</template>

<script>
/* eslint-disable */
import _ from 'lodash'
import * as d3 from 'd3'
import HelpTooltip from '../components/HelpTooltip'
export default {
  name: 'PageIndex',
  components: {
    HelpTooltip
  },
  data () {
    return {
      monthlySaving: 100,
      startAmount: 0,
      savingTarget: 10000,
      yearlyReturn: 5,
      monthlyExpenses: 2000,
      targetOrExpenses: 'expenses',
      increaseMonthlyEvery: 0,
      increaseMonthlyBy: 50,
      returnsMode: 'yearly',
      chartElems: {x:null,y:null,svg:null,xAxis:null,yAxis:null}
    }
  },
  computed: {
    target () {
      return this.targetOrExpenses === 'target' ? this.savingTarget : (this.monthlyExpenses * 12) / (this.yearlyReturn * 0.01)
    },
    statusEveryMonth () {
      // check against infinite while loop (if start with 0 and no ways to increase that in absolute terms, no matter how high the interest it will never reach target)
      if (this.monthlySaving == 0 && this.startAmount == 0 && (this.increaseMonthlyEvery == 0 || this.increaseMonthlyBy == 0)) return []
      var i = 0
      var saved = this.startAmount
      var mSave = this.monthlySaving
      var status = []
      while (saved < this.target) {
        i += 1
        if (this.increaseMonthlyEvery > 0 && i % this.increaseMonthlyEvery === 0) mSave += this.increaseMonthlyBy
        saved += mSave
        if (i % 12 == 0 && this.returnsMode === 'yearly') saved *= (1 + this.yearlyReturn * 0.01)
        else if (i % 4 == 0 && this.returnsMode === 'quarterly') saved *= (1 + this.yearlyReturn * 0.01 * 0.25)
        status.push(saved)
      }
      return status
    }
  },
  methods: {
    redrawChart: _.debounce(function () {
      // Create the X axis:
      var dur = 150
      var {x,y,svg,xAxis,yAxis} = this.chartElems
      var data = this.statusEveryMonth
      x.domain([0, data.length]);
      svg.selectAll(".myXaxis").transition()
        .duration(dur)
        .call(xAxis);

      // create the Y axis
      y.domain([0, d3.max(data, function(d) { return d  }) ]);
      svg.selectAll(".myYaxis")
        .transition()
        .duration(dur)
        .call(yAxis);

      // Create a update selection: bind to the new data
      var u = svg.selectAll(".lineTest")
        .data([data], function(d,i){ return d });
      u.exit().remove()
      // Updata the line
      u
        .enter()
        .append("path")
        .attr("class","lineTest")
        .merge(u)
        .transition()
        .duration(dur)
        .attr("d", d3.line()
          .x(function(d,i) { return x(i) })
          .y(function(d) { return y(d) }))
          .attr("fill", "none")
          .attr("stroke", "steelblue")
          .attr("stroke-width", 2.5)
    }, 300)
  },
  watch: {
    statusEveryMonth (newVal) {
      this.redrawChart()
    }
  },
  mounted () {
    // set the dimensions and margins of the graph
    var margin = {top: 10, right: 30, bottom: 30, left: 50},
        width = 460 - margin.left - margin.right,
        height = 400 - margin.top - margin.bottom;

    // append the svg object to the body of the page
    var svg = d3.select(".saving-forecast")
      .append("svg")
        .attr("width", width + margin.left + margin.right)
        .attr("height", height + margin.top + margin.bottom)
      .append("g")
        .attr("transform",
              "translate(" + margin.left + "," + margin.top + ")");

    // Initialise a X axis:
    var x = d3.scaleLinear().range([0,width]);
    var xAxis = d3.axisBottom().scale(x);
    svg.append("g")
      .attr("transform", "translate(0," + height + ")")
      .attr("class","myXaxis")

    // Initialize an Y axis
    var y = d3.scaleLinear().range([height, 0]);
    var yAxis = d3.axisLeft().scale(y);
    svg.append("g")
      .attr("class","myYaxis")
    this.chartElems.x = x
    this.chartElems.y = y
    this.chartElems.xAxis = xAxis
    this.chartElems.yAxis = yAxis
    this.chartElems.svg = svg
    this.redrawChart()
  }
}
</script>
<style lang="scss">
.slider-sidebar {
  display:flex;
  flex-direction:column;
  width: 400px;

  .slider {
    display:flex;
    flex-direction:column;
  }
}
.saving-forecast {
  display:flex;
  flex-direction:column;
  width: 460px;
  justify-content: center;
  text-align: center;
  margin-left: 30px;
}
</style>
