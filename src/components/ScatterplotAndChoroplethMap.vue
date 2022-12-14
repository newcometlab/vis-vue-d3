<template>
  <div class="row">
    <div class="col-md-6">
      <div class="vis-component" ref="scatterplotChart">
        <svg id="scatter-svg" :width="scatterWidth" :height="scatterHeight" ref="scatterSvg">
          <g class="chart-group" ref="chartGroup">
            <g class="rect-group" ref="rectGroup"></g>
            <g class="brush" ref="brush"></g>
            <g class="axis axis-x" ref="axisX"></g>
            <g class="axis axis-y" ref="axisY"></g>
            <g class="circle-group" ref="circleGroup"></g>
          </g>
        </svg>
      </div>
    </div>
    <div class="col-md-6">
      <div class="vis-component" ref="choroplethChart">
        <svg id="chropleth-svg" :width="chroplethWidth" :height="chroplethHeight" ref="chroplethSvg">
          <g class="usmap" ref="usmap"></g>
        </svg>
      </div>
    </div>
  </div>
</template>

<script>
import * as d3 from "d3";

export default {
  name: "ScatterplotAndChoroplethMap",
  props: {},
  data() {
    return {
      scatterWidth: 0,
      scatterHeight: 400,
      chroplethWidth: 0,
      chroplethHeight: 400,
      scatterPadding: {
        top: 10,
        right: 25,
        bottom: 40,
        left: 75,
      },
      filteredRatesAndIncomes: [],
      isClickedOnState: false,
    };
  },
  mounted() {
    this.drawChart();
  },
  computed: {
    selectedYear: {
      get() {
        return this.$store.getters.selectedYear;
      },
    },
    ratesAndIncomes: {
      get() {
        return this.$store.getters.ratesAndIncomes;
      },
    },
    usStatesGeo: {
      get() {
        return this.$store.getters.usStatesGeo;
      },
    },
    xScaleRange() {
      return [
        0,
        this.scatterWidth -
        this.scatterPadding.left -
        this.scatterPadding.right,
      ];
    },
    yScaleRange() {
      return [
        this.scatterHeight -
        this.scatterPadding.top -
        this.scatterPadding.bottom,
        0,
      ];
    },
    rateRange() {
      if (this.filteredRatesAndIncomes.length === 0) return [0, 1];
      return d3.extent(this.filteredRatesAndIncomes, function (d) {
        return d.rate;
      });
    },
    incomeRange() {
      if (this.filteredRatesAndIncomes.length === 0) return [0, 1];
      return d3.extent(this.filteredRatesAndIncomes, function (d) {
        return d.income;
      });
    },
    xScaleDomain() {
      const x = d3
        .scaleLinear()
        .domain(this.rateRange)
        .nice();
      return x.domain();
    },

    yScaleDomain() {
      const y = d3
        .scaleLinear()
        .domain(this.incomeRange)
        .nice();
      return y.domain();
    },
  },
  watch: {
    selectedYear: {
      handler() {
        this.drawChart();
      },
      deep: true,
    },
    ratesAndIncomes: {
      handler(newValue, oldValue) {
        this.filteredRatesAndIncomes = newValue;
        if (oldValue.length === 0) this.drawChart();
        else this.updateChart();
      },
      deep: true,
    },
    usStatesGeo: {
      handler(_newValue, oldValue) {
        if (!oldValue.features) this.drawChart();
        else this.updateChart();
      },
      deep: true,
    },
  },
  methods: {
    xScale(val) {
      const x = d3
        .scaleLinear()
        .range(this.xScaleRange)
        .domain(this.rateRange)
        .nice();
      return x(val);
    },
    yScale(val) {
      const y = d3
        .scaleLinear()
        .range(this.yScaleRange)
        .domain(this.incomeRange)
        .nice();
      return y(val);
    },
    xScaleInvert(val) {
      const x = d3
        .scaleLinear()
        .range(this.xScaleRange)
        .domain(this.rateRange)
        .nice();
      return x.invert(val);
    },
    yScaleInvert(val) {
      const y = d3
        .scaleLinear()
        .range(this.yScaleRange)
        .domain(this.incomeRange)
        .nice();
      return y.invert(val);
    },
    bivariateColor(xVal, yVal) {
      const [rA, gA, bA] = xVal.match(/\w\w/g).map((c) => parseInt(c, 16));
      const [rB, gB, bB] = yVal.match(/\w\w/g).map((c) => parseInt(c, 16));
      const r = Math.round(rA + (rB - rA) / 2)
        .toString(16)
        .padStart(2, "0");
      const g = Math.round(gA + (gB - gA) / 2)
        .toString(16)
        .padStart(2, "0");
      const b = Math.round(bA + (bB - bA) / 2)
        .toString(16)
        .padStart(2, "0");
      return "#" + r + g + b;
    },
    onBrush(x0, x1, y0, y1) {
      var clear = x0 === x1 || y0 === y1;
      this.filteredRatesAndIncomes = this.ratesAndIncomes.map(function (d) {
        const filtered = clear
          ? false
          : d.rate < x0 || d.rate > x1 || d.income < y0 || d.income > y1;
        return {
          ...d,
          filtered,
          selected: false,
        };
      });
      this.updateChart();
    },
    onSelectState(state) {
      this.filteredRatesAndIncomes = this.ratesAndIncomes.map(function (d) {
        const filtered = !(d.state === state)
        return {
          ...d,
          filtered,
          selected: !filtered
        };
      });
      this.updateChart();
    },
    onDeselectState() {
      this.filteredRatesAndIncomes = this.ratesAndIncomes.map(function (d) {
        return {
          ...d,
          filtered: false,
          selected: false
        };
      });
      this.updateChart();
    },
    drawChart() {
      if (this.$refs.scatterplotChart)
        this.scatterWidth = this.$refs.scatterplotChart.clientWidth;
      if (this.$refs.choroplethChart) {
        this.chroplethWidth = this.$refs.choroplethChart.clientWidth;
        this.chroplethHeight = this.$refs.choroplethChart.clientWidth * 0.6;
      }

      d3.select(this.$refs.chartGroup).attr(
        "transform",
        `translate(${this.scatterPadding.left},${this.scatterPadding.top})`
      );

      this.drawScatterXAxis();
      this.drawScatterYAxis();
      this.drawScatterRects();
      this.drawScatterBrush();
      this.drawScatterPlots();
      this.drawUsmap();
    },
    updateChart() {
      this.drawScatterRects();
      this.drawScatterBrush();
      this.drawScatterPlots();
      this.drawUsmap();
    },
    drawScatterXAxis() {
      const x = d3.scaleLinear().range(this.xScaleRange);
      x.domain(this.rateRange).nice();
      const axisX = d3.select(this.$refs.axisX);
      axisX
        .attr(
          "transform",
          `translate( 0, ${this.scatterHeight -
          this.scatterPadding.top -
          this.scatterPadding.bottom} )`
        )
        .call(
          d3.axisBottom().scale(x)
            .tickFormat(d3.format(".2s"))
        )

      axisX
        .selectAll('text.axisXTitle')
        .data(["Burglary Rate (per 100.000 people)"])
        .enter()
        .append("text")
        .attr('class', 'axisXTitle')
        .attr(
          "x",
          this.scatterWidth -
          this.scatterPadding.left -
          this.scatterPadding.right
        )
        .attr("y", 35)
        .style("text-anchor", "end")
        .style("fill", "#000")
        .style("font-weight", "bold")
        .text(d => d);
    },
    drawScatterYAxis() {
      const y = d3.scaleLinear().range(this.yScaleRange);
      y.domain(this.incomeRange).nice();
      const axisY = d3.select(this.$refs.axisY);
      axisY
        .call(
          d3.axisLeft().scale(y)
            .tickFormat(d3.format("$.2s"))
        )
      axisY
        .selectAll('text.axisYTitle')
        .data(["Per Capita Disposable Personal Income (in $)"])
        .enter()
        .append("text")
        .attr("transform", "rotate(-90)")
        .attr('class', 'axisYTitle')
        .attr("x", 0)
        .attr("y", -40)
        .style("text-anchor", "end")
        .style("fill", "#000")
        .style("font-weight", "bold")
        .text(d => d);
    },
    drawScatterPlots() {
      if (this.filteredRatesAndIncomes.length === 0) return;

      var vm = this;
      const xDomainStep = (this.xScaleDomain[1] - this.xScaleDomain[0]) / 4;
      const yDomainStep = (this.yScaleDomain[1] - this.yScaleDomain[0]) / 4;
      const xColor = d3
        .scaleThreshold()
        .domain([
          this.xScaleDomain[0] + xDomainStep,
          this.xScaleDomain[0] + 2 * xDomainStep,
          this.xScaleDomain[0] + 3 * xDomainStep,
        ])
        .range(["#e8e8e8", "#aed6e0", "#73cfe6", "#37c3e6"]);
      const yColor = d3
        .scaleThreshold()
        .domain([
          this.yScaleDomain[0] + yDomainStep,
          this.yScaleDomain[0] + 2 * yDomainStep,
          this.yScaleDomain[0] + 3 * yDomainStep,
        ])
        .range(["#fbb4b9", "#f768a1", "#c51b8a", "#7a0177"]);

      const circles = d3.select(this.$refs.circleGroup);

      circles
        .selectAll("circle")
        .data(this.filteredRatesAndIncomes)
        .join("circle")
        .attr("r", 4)
        .style("stroke", "#fff")
        .attr("cx", function (d) {
          return vm.xScale(d.rate);
        })
        .attr("cy", function (d) {
          return vm.yScale(d.income);
        })
        .style("fill", function (d) {
          return vm.bivariateColor(xColor(d.rate), yColor(d.income));
        })
        .style("opacity", 1)
        .style("stroke-width", 2)

      circles
        .selectAll("circle")
        .append('title')
        .text(d => d.state);
      circles
        .selectAll("circle")
        .on('mouseover', function () {
          d3.select(this).attr("r", 8);
        })
        .on('mouseout', function () {
          d3.select(this).attr("r", 4);
        });
      circles
        .selectAll("circle")
        .style("opacity", function (d) {
          return d.filtered ? 0.5 : 1;
        })
        .style("stroke-width", function (d) {
          return d.filtered ? 1 : 2;
        })
        .attr("r", function (d) {
          return d.selected ? 8 : 4
        });
    },
    drawScatterRects() {
      if (this.filteredRatesAndIncomes.length === 0) return;

      var vm = this;

      const xDomainStep = (this.xScaleDomain[1] - this.xScaleDomain[0]) / 4;
      const yDomainStep = (this.yScaleDomain[1] - this.yScaleDomain[0]) / 4;
      const xColor = d3
        .scaleThreshold()
        .domain([
          this.xScaleDomain[0] + xDomainStep,
          this.xScaleDomain[0] + 2 * xDomainStep,
          this.xScaleDomain[0] + 3 * xDomainStep,
        ])
        .range(["#e8e8e8", "#aed6e0", "#73cfe6", "#37c3e6"]);
      const yColor = d3
        .scaleThreshold()
        .domain([
          this.yScaleDomain[0] + yDomainStep,
          this.yScaleDomain[0] + 2 * yDomainStep,
          this.yScaleDomain[0] + 3 * yDomainStep,
        ])
        .range(["#fbb4b9", "#f768a1", "#c51b8a", "#7a0177"]);

      const xRectPairs = d3.pairs(
        d3.merge([
          [this.xScaleDomain[0]],
          xColor.domain(),
          [this.xScaleDomain[1]],
        ])
      );
      const yRectPairs = d3.pairs(
        d3.merge([
          [this.yScaleDomain[0]],
          yColor.domain(),
          [this.yScaleDomain[1]],
        ])
      );

      xRectPairs.forEach((xd) => {
        yRectPairs.forEach((yd) => {
          d3.select(this.$refs.rectGroup)
            .append("rect")
            .attr("x", vm.xScale(xd[0]))
            .attr("width", vm.xScale(xd[1]) - vm.xScale(xd[0]))
            .attr("y", vm.yScale(yd[1]))
            .attr("height", vm.yScale(yd[0]) - vm.yScale(yd[1]))
            .style("fill", vm.bivariateColor(xColor(xd[0]), yColor(yd[0])));
        });
      });
    },
    drawScatterBrush() {
      const vm = this;

      const brush = d3
        .brush()
        .extent([
          [0, 0],
          [this.scatterWidth, this.scatterHeight],
        ])
        .on("start brush", function (ev) {
          const selection = ev.selection;

          const x0 = vm.xScaleInvert(selection[0][0]);
          const x1 = vm.xScaleInvert(selection[1][0]);
          const y0 = vm.yScaleInvert(selection[1][1]);
          const y1 = vm.yScaleInvert(selection[0][1]);

          vm.onBrush(x0, x1, y0, y1);
        });

      d3.select(this.$refs.brush).call(brush);
    },
    drawUsmap() {
      if (
        !this.usStatesGeo.features ||
        !this.filteredRatesAndIncomes.length === 0
      )
        return;

      const vm = this;

      const xDomainStep = (this.xScaleDomain[1] - this.xScaleDomain[0]) / 4;
      const yDomainStep = (this.yScaleDomain[1] - this.yScaleDomain[0]) / 4;
      const xColor = d3
        .scaleThreshold()
        .domain([
          this.xScaleDomain[0] + xDomainStep,
          this.xScaleDomain[0] + 2 * xDomainStep,
          this.xScaleDomain[0] + 3 * xDomainStep,
        ])
        .range(["#e8e8e8", "#aed6e0", "#73cfe6", "#37c3e6"]);
      const yColor = d3
        .scaleThreshold()
        .domain([
          this.yScaleDomain[0] + yDomainStep,
          this.yScaleDomain[0] + 2 * yDomainStep,
          this.yScaleDomain[0] + 3 * yDomainStep,
        ])
        .range(["#fbb4b9", "#f768a1", "#c51b8a", "#7a0177"]);

      const projection = d3
        .geoAlbersUsa()
        .scale([this.chroplethWidth * 1.25])
        .translate([this.chroplethWidth / 2, this.chroplethHeight / 2]);

      const path = d3.geoPath().projection(projection);

      const chroplethSvg = d3.select(this.$refs.chroplethSvg);
      chroplethSvg
        .on('click', function () {
          if (vm.isClickedOnState === false) vm.onDeselectState();
          vm.isClickedOnState = false;
        })

      const usmap = d3.select(this.$refs.usmap);

      usmap
        .selectAll("path")
        .data(this.usStatesGeo.features)
        .enter()
        .append("path")
        .attr("d", path)
        .style("stroke", "#fff")
        .style("stroke-width", 1)
        .style("fill", "#ddd")
        .append('title')
        .text(d => d.properties.name);

      usmap.selectAll("path")
        .on('click', function (ev) {
          vm.isClickedOnState = true;
          vm.onSelectState(ev.target.__data__.properties.name);
        });
      usmap.selectAll("path").style("fill", function (d) {
        const row = vm.filteredRatesAndIncomes.find(
          (row) => row.state === d.properties.name
        );
        if (!row) return "transparent";
        return row.filtered
          ? "#ddd"
          : vm.bivariateColor(xColor(row.rate), yColor(row.income));
      });
    },
  },
};
</script>

<style>

</style>
