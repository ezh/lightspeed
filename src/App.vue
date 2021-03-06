<template>
  <v-container grid list-md text-xs-center id="app">
    <v-layout row wrap>
      <v-flex xs10 offset-xs1 @keydown.up.prevent="scrollUp" @keydown.down.prevent="scrollDown">
        <v-toolbar class="white search-toolbar" floating dense @keyup.enter="open">
          <v-text-field v-model="searchString" prepend-icon="search" full-width hide-details single-line autofocus></v-text-field>
          <Jumper v-if="debounceIndicator"></Jumper>
        </v-toolbar>
      </v-flex>
    </v-layout>
    <v-layout row wrap>
      <v-flex xs8 offset-xs2>
        <List :tabs="tabs" :bookmarks="bookmarks" :history="filteredHistory" :selected="selectedItemId" ref="list"></List>
      </v-flex>
    </v-layout>
  </v-container>
</template>

<script>
import List from './components/List.vue';
import Jumper from './components/Jumper.vue';

import contains from './util/contains';

export default {
  data() {
    return {
      searchString: '',
      tabs: [],
      bookmarks: [],
      fullHistory: [],
      filteredHistory: [],
      selectedItemId: null,
      selectedItemIndex: -1,
      debounceTimer: null,
      debounceIndicator: false
    };
  },

  computed: {
    consolidatedList() {
      return [...this.tabs, ...this.bookmarks, ...this.filteredHistory];
    }
  },

  methods: {
    scrollUp() {
      if (!this.selectedItemId) {
        return;
      } else {
        if (this.consolidatedList[this.selectedItemIndex - 1]) {
          this.selectedItemIndex--;
          this.selectedItemId = this.consolidatedList[this.selectedItemIndex].id;
        }
      }

      const el = document.getElementsByClassName('selected')[0];
      if ((el.offsetTop - el.offsetHeight) < window.pageYOffset)
        window.scroll(0, el.offsetTop - (el.offsetHeight));
    },

    scrollDown() {
      if (!this.selectedItemId) {
        this.selectedItemIndex = 0;
        this.selectedItemId = this.consolidatedList[this.selectedItemIndex].id;
      } else {
        if (this.consolidatedList[this.selectedItemIndex + 1]) {
          this.selectedItemIndex++;
          this.selectedItemId = this.consolidatedList[this.selectedItemIndex].id;
        }
      }

      const el = document.getElementsByClassName('selected')[0];
      if ((el.offsetTop + (el.offsetHeight * 3)) > (window.pageYOffset + window.innerHeight))
        window.scroll(0, el.offsetTop - window.innerHeight + (el.offsetHeight * 3));
    },

    open() {
      let selectedItem;
      if (!this.selectedItemId) {
        selectedItem = this.consolidatedList[0];
      } else {
        selectedItem = this.consolidatedList[this.selectedItemIndex];
      }

      if ('windowId' in selectedItem) {
        this.$refs.list.focusTab(selectedItem);
      } else {
        this.$refs.list.open(selectedItem);
      }
    }
  },

  mounted() {
    chrome.tabs.query({}, tabs => {
      this.tabs = [];
      for (let i = tabs.length - 1; i >= 0; i--) {
        this.tabs.push(tabs[i]);
        if (this.tabs.length === 30) {
          break;
        }
      }
    });
    chrome.history.search({ text: '', maxResults: 0, startTime: 0 }, history => this.fullHistory = history);
  },

  watch: {
    searchString(newSearchString) {
      clearTimeout(this.debounceTimer);

      this.selectedItemId = null;
      this.selectedItemIndex = -1;

      if (newSearchString === '') {
        this.debounceIndicator = false;
        this.bookmarks = [];
        this.filteredHistory = [];
        chrome.tabs.query({}, tabs => {
          this.tabs = [];
          for (let i = tabs.length - 1; i >= 0; i--) {
            this.tabs.push(tabs[i]);
            if (this.tabs.length === 30) {
              break;
            }
          }
        });
        return;
      }

      this.debounceIndicator = true;

      this.debounceTimer = setTimeout(() => {
        this.debounceIndicator = false;

        const words = newSearchString.toLowerCase().split(' ');

        chrome.tabs.query({}, tabs => {
          this.tabs = [];
          for (let i = tabs.length - 1; i >= 0; i--) {
            if (contains(tabs[i], words)) {
              this.tabs.push(tabs[i]);
              if (this.tabs.length === 30) {
                break;
              }
            }
          }
        });

        chrome.bookmarks.search(newSearchString, bookmarks => {
          for (const bookmark of bookmarks) {
            this.bookmarks.push(bookmark);
            if (this.bookmarks.length === 30) {
              break;
            }
          }
        });

        this.filteredHistory = [];
        for (const item of this.fullHistory) {
          if (contains(item, words)) {
            this.filteredHistory.push(item);
            if (this.filteredHistory.length === 30) {
              break;
            }
          }
        }
      }, 250);
    }
  },

  components: { List, Jumper }
}
</script>

<style lang="stylus">
@import '../node_modules/vuetify/src/stylus/main'

#app
  margin-top 20px

.search-toolbar
  width 50% !important

  .toolbar__content
    width 100%
</style>
