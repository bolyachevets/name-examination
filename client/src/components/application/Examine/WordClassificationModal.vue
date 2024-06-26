<!--eslint-disable-->
<template>
  <v-dialog persistent
            v-on:keydown="handleKeydown"
            no-click-animation
            width="760"
            content-class="word-classification-modal"
            v-model="modalVisible">
    <v-container>
      <v-layout column align-end>
        <v-flex>
          <v-btn class="ma-0 pa-0"
                 @click="showConfirmClose = true"
                 flat>
            <v-icon color="light-blue">close</v-icon>
            <span color="blue">close</span>
          </v-btn>
        </v-flex>
      </v-layout>
      <template v-if="!showConfirmClose">
        <v-layout column>
          <v-flex class="fs-24">Word Classification</v-flex>
          <v-flex class="my-3">
            Classify appropriate words as either Distinctive or Descriptive. Do not classify designations, stopwords
            or words that that you know have already been classified.
          </v-flex>
        </v-layout>
        <v-layout column align-center>
          <v-flex class="name-text mt-2 mb-4">
          <span :class="focus === i ? 'highlight' : ''"
                :key="i+word"
                class="mx-1"
                v-for="(word, i) in chunkedName">{{ word}}</span></v-flex>
        </v-layout>
        <v-layout hairline pb-1>
          <v-flex lg6 />
          <v-flex lg3 yellow-header><span class="ul">D</span>istinctive</v-flex>
          <v-flex lg3 blue-header>De<span class="ul">s</span>criptive</v-flex>
        </v-layout>
        <v-layout :class="i === focus ? 'focused' : ''"
                  :key="word+i"
                  class="word-column hairline"
                  v-for="(word, i) in chunkedName">
          <v-flex lg6>{{ word }}</v-flex>
          <v-flex lg3><v-checkbox value="DIST"
                                  v-model="classifications[i]" /></v-flex>
          <v-flex lg3><v-checkbox value="DESC"
                                  v-model="classifications[i]" /></v-flex>
        </v-layout>
        <v-layout column align-end mt-4>
          <v-flex>
            <v-btn @click="showConfirmClose = true"
                   class="mx-1 pa-0 action-button"
                   flat><img src="/static/images/buttons/cancel.png" />
            </v-btn>
            <v-btn class="mx-1 pa-0 action-button"
                   @click="submit()"
                   flat><img src="/static/images/buttons/green-done.png" /></v-btn>
          </v-flex>
        </v-layout>
      </template>
      <template v-else>
        <v-layout column>
          <v-flex class="fs-24">Clear Classifications?</v-flex>
          <v-flex class="my-3">You have classified some words in this name.  Cancelling will clear these
          classifications.</v-flex>
        </v-layout>
        <v-layout column align-center mt-4>
          <v-flex>
            <v-btn class="mx-1 pa-0 action-button"
                   @click="modalVisible = false"
                   flat><img src="/static/images/buttons/clear-classifications.png" />
            </v-btn>
            <v-btn class="mx-1 pa-0 action-button"
                   @click="showConfirmClose = false"
                   flat><img src="/static/images/buttons/return-classifications.png" />
            </v-btn>
          </v-flex>
        </v-layout>
      </template>
    </v-container>
  </v-dialog>
</template>

<script>
/* eslint-disable */
import Vue from 'vue'
import { mapState } from 'vuex'

export default {
  name: 'WordClassificationModal',
  data() {
    return {
      showConfirmClose: false,
      focus: 0,
      classifications: {
        1: ''
      }
    }
  },
  watch: {
    modalVisible(newVal) {
      if (newVal) {
        this.classifications = {1: ''}
        this.showConfirmClose = false
        this.focus = 0
      }
    },
    showConfirmClose(newVal) {
      if (newVal) {
        let keys = Object.keys(this.classifications)
        for (let key of keys) {
          if (this.classifications[key]) {
            return
          }
        }
      newVal = false
      this.modalVisible = false
      }
    }
  },
  computed: {
    ...mapState({
      wordClassificationModalVisible: state => state.wordClassificationModalVisible,
      name: state => state.wordClassificationModalName,
      userId: state => state.userId
    }),
    chunkedName() {
      return this.name.split(' ')
    },
    modalVisible: {
      get() {
        return this.wordClassificationModalVisible
      },
      set(value) {
        this.$store.commit('toggleWordClassificationModal', value)
      }
    }
  },
  methods: {
    cancel() {
      this.modalVisible = false
      return
    },
    handleClick(i, type) {
      this.classifications[i] = type
    },
    handleKeydown(event) {
      let capturedKeyCodes = ['KeyD', 'KeyS', 'ArrowUp', 'ArrowDown', 'Space', 'Enter', 'Escape']
      if (!capturedKeyCodes.includes(event.code)) {
        return event
      }
      event.preventDefault()
      event.stopPropagation()
      let max = this.chunkedName.length
      let moveDown = () => {
        if (this.focus < max) {
          this.focus++
        }
        if (this.focus >= max) {
          this.focus = 0
        }
      }
      let moveUp = () => {
        if (this.focus === 0) {
          this.focus = max - 1
        } else {
          this.focus--
        }
      }
      if (this.showConfirmClose) {
        if (event.code === 'Enter') {
          this.modalVisible = false
          return
        }
        if (event.code === 'Escape') {
          this.showConfirmClose = false
          return
        }
        return event
      }
      switch (event.code) {
        case 'ArrowDown':
         moveDown()
          break
        case 'ArrowUp':
          moveUp()
          break
        case 'KeyD':
          Vue.set(
            this.classifications,
            this.focus,
            'DIST'
          )
          moveDown()
          return
        case 'KeyS':
          Vue.set(
            this.classifications,
            this.focus,
            'DESC'
          )
          moveDown()
          return
        case 'Enter':
          this.submit()
          return
        case 'Escape':
          this.showConfirmClose = true
          return
        case 'Space':
          Vue.set(
            this.classifications,
            this.focus,
            ''
          )
          moveDown()
      }
    },
    submit() {
      if (this.classifications.length === 0) {
        this.modalVisible = false
        return
      }
      let promises = []
      for (let i in this.classifications) {
        if (this.classifications[i]) {
          let payload = {
            url: this.chunkedName[i],
            data: {
              classification: this.classifications[i],
              examiner: this.userId,
              name: this.name,
              word: this.chunkedName[i]
            }
          }
          let resp = this.$store.dispatch('postWordForClassification', payload)
          promises.push(resp)
        }
      }
      // resolve all promises (ie, make multiple calls to API) concurrently
      Promise.all(promises).then( () => {
        this.modalVisible = false
      }).catch(err => {
        console.log(err)
        this.modalVisible = false
      })
    }
  }
}
</script>

<style scoped>
  .name-text {
    text-transform: uppercase;
    color: var(--d-blue);
    font-size: 19px;
    font-weight: 700;
  }
  .hairline {
    border-bottom: 1px solid lightgray !important;
  }
  .word-column {
    height: 50px;
    font-size: 19px;
    padding: 5px 0 5px 0;
  }
  .highlight {
    color: var(--gold) !important;
  }
  .focused {
    background-color: var(--l-blue);
  }
  .yellow-header {
    font-weight: 700;
    color: var(--d-blue);
    font-size: 16px;
  }
  .blue-header {
    font-weight: 700;
    color: var(--gold);
    font-size: 16px;
  }
  .ul {
    text-decoration: underline;
  }
</style>
