<template>
    <div class="wrapper">
        <div class="container">
            <div class="slider-track" :style="{'background': background}"></div>
            <input type="range" min="0" max="1000" value="0" :id="slider1Id" @change="slideOne()">
            <input type="range" min="0" max="1000" value="0" :id="slider2Id" @change="slideTwo()">
        </div>
    </div>
</template>

<script>
export default {
  name: 'Slider',
  data () {
    return {
      valueLeft: 0,
      valueRight: 0,
      background: '',
      minGap: 10,
      slider1Id: '',
      slider2Id: ''
    }
  },
  props: {
    maxValue: {
      type: Number
    }
  },
  // computed: {
  //   selectedMaxValue () {
  //     return this.maxValue
  //   },
  // },
  methods: {
    slideOne () {
      const sliderOne = document.getElementById(this.slider1Id)
      if (this.valueRight - parseInt(sliderOne.value) <= this.minGap) {
        sliderOne.value = this.valueRight - this.minGap
        this.valueLeft = this.valueRight - this.minGap
      } else {
        this.valueLeft = parseInt(sliderOne.value)
      }
      this.fillColor()
      this.$emit('change-slider-value', { valueLeft: this.valueLeft, valueRight: this.valueRight })
    },

    slideTwo () {
      const sliderTwo = document.getElementById(this.slider2Id)
      if (parseInt(sliderTwo.value) - this.valueLeft <= this.minGap) {
        sliderTwo.value = this.valueLeft + this.minGap
        this.valueRight = this.valueLeft + this.minGap
      } else {
        this.valueRight = parseInt(sliderTwo.value)
      }
      this.fillColor()
      this.$emit('change-slider-value', { valueLeft: this.valueLeft, valueRight: this.valueRight })
    },

    fillColor () {
      const percent1 = (this.valueLeft / this.maxValue) * 100
      const percent2 = (this.valueRight / this.maxValue) * 100
      this.background = `linear-gradient(to right, #dadae5 ${percent1}% , #3264fe ${percent1}% , #3264fe ${percent2}%, #dadae5 ${percent2}%)`
    }
  },
  created () {
    this.slider1Id = 'slider1' + Math.floor(Math.random() * (1000))
    this.slider2Id = 'slider2' + Math.floor(Math.random() * (1000))
  },
  watch: {
    maxValue: function () {
      const sliderTwo = document.getElementById(this.slider2Id)
      const sliderOne = document.getElementById(this.slider1Id)
      if (this.maxValue !== undefined) {
        sliderTwo.max = this.maxValue
        // sliderTwo.value = this.maxValue
        sliderOne.max = this.maxValue
        // this.valueRight = this.maxValue
        // this.valueLeft = 0
        // this.background = 'linear-gradient(to right, #dadae5 0% , #3264fe 0% , #3264fe 100%, #dadae5 100%)'
        // this.$emit('change-slider-value', { valueLeft: this.valueLeft, valueRight: this.valueRight })
      }
    }
  }
}
</script>

<style>
.wrapper{
    position: relative;
    width: 95vmin;
    background-color: #ffffff;
    padding: 50px 40px 20px 40px;
    border-radius: 10px;
}
.container{
    position: relative;
    width: 100%;
    height: 100px;
    margin-top: 30px;
}
input[type="range"]{
    -webkit-appearance: none;
    -moz-appearance: none;
    appearance: none;
    width: 100%;
    outline: none;
    position: absolute;
    margin: auto;
    top: 0;
    bottom: 0;
    background-color: transparent;
    pointer-events: none;
}
.slider-track{
    width: 100%;
    height: 5px;
    position: absolute;
    margin: auto;
    top: 0;
    bottom: 0;
    border-radius: 5px;
}
input[type="range"]::-webkit-slider-runnable-track{
    -webkit-appearance: none;
    height: 5px;
}
input[type="range"]::-moz-range-track{
    -moz-appearance: none;
    height: 5px;
}
input[type="range"]::-ms-track{
    appearance: none;
    height: 5px;
}
input[type="range"]::-webkit-slider-thumb{
    -webkit-appearance: none;
    height: 1.7em;
    width: 1.7em;
    background-color: #3264fe;
    cursor: pointer;
    margin-top: -9px;
    pointer-events: auto;
    border-radius: 50%;
}
input[type="range"]::-moz-range-thumb{
    -webkit-appearance: none;
    height: 1.7em;
    width: 1.7em;
    cursor: pointer;
    border-radius: 50%;
    background-color: #3264fe;
    pointer-events: auto;
}
input[type="range"]::-ms-thumb{
    appearance: none;
    height: 1.7em;
    width: 1.7em;
    cursor: pointer;
    border-radius: 50%;
    background-color: #3264fe;
    pointer-events: auto;
}
input[type="range"]:active::-webkit-slider-thumb{
    background-color: #ffffff;
    border: 3px solid #3264fe;
}
</style>
