<template>
  <div class="vue-form-wizard">
    <div class="wizard-header">
      <slot name="title">
        <h4 class="wizard-title">{{title}}</h4>
        <p class="category">{{subtitle}}</p>
      </slot>
    </div>
    <div class="wizard-navigation">
      <div class="wizard-progress-with-circle">
        <div class="wizard-progress-bar"
             :style="progressBarStyle"></div>
      </div>
      <ul class="wizard-nav wizard-nav-pills">
        <slot name="step" v-for="(tab, index) in tabs"
              :tab="tab"
              :index="index"
              :navigate-to-tab="navigateToTab"
              :transition="transition">
          <wizard-step :tab="tab"
                       @click.native="navigateToTab(index)"
                       :transition="transition"
                       :index="index">
          </wizard-step>
        </slot>
      </ul>
      <div class="wizard-tab-content">
        <slot>
        </slot>
      </div>
    </div>

    <div class="wizard-card-footer clearfix" v-if="!hideButtons">
      <slot name="footer"
            :next-tab="nextTab"
            :prev-tab="prevTab"
            :active-tab-index="activeTabIndex"
            :is-last-step="isLastStep"
            :fill-button-style="fillButtonStyle">

        <template>
        <span @click="prevTab" v-if="displayPrevButton" class="wizard-footer-left">
          <slot name="prev">
            <wizard-button :style="fillButtonStyle"
                    :disabled="loading">
              {{backButtonText}}
            </wizard-button>
          </slot>
        </span>
        </template>

        <template>
         <span @click="finish" class="wizard-footer-right" v-if="isLastStep">
           <slot name="finish">
             <wizard-button :style="fillButtonStyle">
              {{finishButtonText}}
            </wizard-button>
          </slot>
        </span>
        </template>

        <template>
        <span @click="nextTab" class="wizard-footer-right" v-if="!isLastStep">
         <slot name="next">
           <wizard-button :style="fillButtonStyle"
                   :disabled="loading">
            {{nextButtonText}}
           </wizard-button>
         </slot>
       </span>
        </template>
      </slot>
    </div>
  </div>
</template>
<script>
  import WizardButton from './WizardButton.vue'
  import WizardStep from './WizardStep.vue'
  export default{
    name: 'form-wizard',
    components: {
      WizardButton,
      WizardStep
    },
    props: {
      title: {
        type: String,
        default: 'Awesome Wizard'
      },
      subtitle: {
        type: String,
        default: 'Split a complicated flow in multiple steps'
      },
      nextButtonText: {
        type: String,
        default: 'Next'
      },
      backButtonText: {
        type: String,
        default: 'Back'
      },
      finishButtonText: {
        type: String,
        default: 'Finish'
      },
      hideButtons: {
        type: Boolean,
        default: false
      },
      validateOnBack: Boolean,
      /***
       * Applies to text, border and circle
       */
      color: {
        type: String,
        default: '#e74c3c'
      },
      errorColor: {
        type: String,
        default: '#8b0000'
      },
      shape: {
        type: String,
        default: 'circle'
      },
      /**
       * Name of the transition when transition between steps
       * */
      transition: {
        type: String,
        default: ''
      },
      /***
       *
       * Index of the initial tab to display
       */
      startIndex: {
        type: Number,
        default: 0,
        validator: (value) => {
          return value >= 0
        }
      }
    },
    data () {
      return {
        activeTabIndex: 0,
        isLastStep: false,
        currentPercentage: 0,
        maxStep: 0,
        loading: false,
        tabs: []
      }
    },
    computed: {
      tabCount () {
        return this.tabs.length
      },
      displayPrevButton () {
        return this.activeTabIndex !== 0
      },
      stepPercentage () {
        return 1 / (this.tabCount * 2) * 100
      },
      progressBarStyle () {
        return {
          backgroundColor: this.color,
          width: `${this.progress}%`,
          color: this.color
        }
      },
      fillButtonStyle () {
        return {
          backgroundColor: this.color,
          borderColor: this.color,
          color: 'white'
        }
      },
      progress () {
        let percentage = 0
        if (this.activeTabIndex > 0) {
          let stepsToAdd = 1
          let stepMultiplier = 2
          percentage = this.stepPercentage * ((this.activeTabIndex * stepMultiplier) + stepsToAdd)
        } else {
          percentage = this.stepPercentage
        }
        return percentage
      }
    },
    methods: {
      addTab (item) {
        const index = this.$slots.default.indexOf(item.$vnode)
        this.tabs.splice(index, 0, item)
        // if a step is added before the current one, go to it
        if (index < this.activeTabIndex + 1) {
          this.maxStep = index
          this.changeTab(this.activeTabIndex + 1, index)
        }
      },
      removeTab (item) {
        const tabs = this.tabs
        const index = tabs.indexOf(item)
        if (index > -1) {
          // Go one step back if the current step is removed
          if (index === this.activeTabIndex) {
            this.maxStep = this.activeTabIndex - 1
            this.changeTab(this.activeTabIndex, this.activeTabIndex - 1)
          }
          if (index < this.activeTabIndex) {
            this.maxStep = this.activeTabIndex - 1
            this.activeTabIndex = this.activeTabIndex - 1
          }
          tabs.splice(index, 1)
        }
      },
      navigateToTab (index) {
        this.$emit('on-change', this.activeTabIndex, index)
        let validate = index > this.activeTabIndex
        if (index <= this.maxStep) {
          let cb = () => {
            this.changeTab(this.activeTabIndex, index)
          }
          if (validate) {
            this.beforeTabChange(this.activeTabIndex, cb)
          } else {
            this.setValidationError(null)
            cb()
          }
        }
      },
      nextTab () {
        this.$emit('on-change', this.activeTabIndex, this.activeTabIndex + 1)
        let cb = () => {
          if (this.activeTabIndex < this.tabCount - 1) {
            this.changeTab(this.activeTabIndex, this.activeTabIndex + 1)
          } else {
            this.isLastStep = true
            this.$emit('finished')
          }
        }
        this.beforeTabChange(this.activeTabIndex, cb)
      },
      prevTab () {
        this.$emit('on-change', this.activeTabIndex, this.activeTabIndex - 1)
        let cb = () => {
          if (this.activeTabIndex > 0) {
            this.setValidationError(null)
            this.changeTab(this.activeTabIndex, this.activeTabIndex - 1)
            this.isLastStep = false
          }
        }
        if (this.validateOnBack) {
          this.beforeTabChange(this.activeTabIndex, cb)
        } else {
          cb()
        }
      },
      finish () {
        let cb = () => {
          this.$emit('on-complete')
        }
        this.beforeTabChange(this.activeTabIndex, cb)
      },
      setLoading (value) {
        this.loading = value
        this.$emit('on-loading', value)
      },
      setValidationError (error) {
        this.tabs[this.activeTabIndex].validationError = error
        this.$emit('on-error', error)
      },
      validateBeforeChange (promiseFn, callback) {
        this.setValidationError(null)
        // we have a promise
        if (promiseFn.then && typeof promiseFn.then === 'function') {
          this.setLoading(true)
          promiseFn.then((res) => {
            this.setLoading(false)
            let validationResult = res === true
            this.executeBeforeChange(validationResult, callback)
          }).catch((error) => {
            this.setLoading(false)
            this.setValidationError(error)
          })
          // we have a simple function
        } else {
          let validationResult = promiseFn === true
          this.executeBeforeChange(validationResult, callback)
        }
      },
      executeBeforeChange (validationResult, callback) {
        this.$emit('on-validate', validationResult, this.activeTabIndex)
        if (validationResult) {
          callback()
        } else {
          this.tabs[this.activeTabIndex].validationError = 'error'
        }
      },
      beforeTabChange (index, callback) {
        if (this.loading) {
          return
        }
        let oldTab = this.tabs[index]
        if (oldTab && oldTab.beforeChange !== undefined) {
          let tabChangeRes = oldTab.beforeChange()
          this.validateBeforeChange(tabChangeRes, callback)
        } else {
          callback()
        }
      },
      changeTab (oldIndex, newIndex) {
        let oldTab = this.tabs[oldIndex]
        let newTab = this.tabs[newIndex]
        if (oldTab) {
          oldTab.active = false
        }
        if (newTab) {
          newTab.active = true
        }
        this.activeTabIndex = newIndex
        this.activateTabAndCheckStep(this.activeTabIndex)
        return true
      },
      tryChangeRoute (tab) {
        if (this.$router && tab.route) {
          this.$router.push(tab.route)
        }
      },
      checkStep () {
        if (this.activeTabIndex === this.tabCount - 1) {
          this.isLastStep = true
        } else {
          this.isLastStep = false
        }
      },
      increaseMaxStep () {
        if (this.activeTabIndex > this.maxStep) {
          this.maxStep = this.activeTabIndex
        }
      },
      checkRouteChange (route) {
        let matchingTabIndex = -1
        let matchingTab = this.tabs.find((tab, index) => {
          let match = tab.route === route
          if (match) {
            matchingTabIndex = index
          }
          return match
        })

        if (matchingTab && !matchingTab.active) {
          const shouldValidate = matchingTabIndex > this.activeTabIndex
          this.navigateToTab(matchingTabIndex, shouldValidate)
        }
      },
      deactivateTabs () {
        this.tabs.forEach(tab => {
          tab.active = false
        })
      },
      activateTab (index) {
        this.deactivateTabs()
        let tab = this.tabs[index]
        tab.active = true
        tab.checked = true
        this.tryChangeRoute(tab)
      },
      activateTabAndCheckStep (index) {
        this.activateTab(index)
        this.checkStep()
        if (index > this.maxStep) {
          this.maxStep = index
        }
        this.activeTabIndex = index
      },
      initializeTabs () {
        if (this.tabs.length > 0 && this.startIndex === 0) {
          this.activateTab(this.activeTabIndex)
        }
        if (this.startIndex < this.tabs.length) {
          this.activateTabAndCheckStep(this.startIndex)
        } else {
          window.console.warn(`Prop startIndex set to ${this.startIndex} is greater than the number of tabs - ${this.tabs.length}. Make sure that the starting index is less than the number of tabs registered`)
        }
      }
    },
    mounted () {
      this.initializeTabs()
    },
    watch: {
      '$route.path': function (newRoute) {
        this.checkRouteChange(newRoute)
      }
    }
  }
</script>
<style lang="scss">
  @import "./../assets/wizard";
</style>
