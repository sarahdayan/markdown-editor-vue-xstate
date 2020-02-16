<template>
  <div id="app">
    <div class="container">
      <textarea class="editor" v-model="content" spellcheck="false" />
      <div
        v-show="current.matches('visible.rendered')"
        class="render markdown-body"
        v-html="rendered"
      />
      <pre v-show="current.matches('visible.raw')" class="render raw">{{
        raw
      }}</pre>
    </div>
    <div class="toggle">
      <button
        :aria-label="
          current.matches('visible.raw')
            ? 'Show rendered Markdown'
            : 'Show HTML code'
        "
        class="btn"
        v-show="current.matches('visible')"
        @click="send('SWAP')"
      >
        <menu-icon v-show="current.matches('visible.raw')" />
        <code-icon v-show="current.matches('visible.rendered')" />
      </button>
      <button
        :aria-label="
          current.matches('hidden') ? 'Show preview' : 'Hide preview'
        "
        class="btn"
        @click="send('TOGGLE')"
      >
        <eye-icon v-show="current.matches('hidden')" />
        <eye-off-icon v-show="current.matches('visible')" />
      </button>
    </div>
  </div>
</template>

<script>
import { Machine, State, interpret } from 'xstate';
import MarkdownIt from 'markdown-it';
import { indent } from 'indent.js';
import debounce from 'lodash.debounce';

import EyeIcon from './assets/icons/eye.svg';
import EyeOffIcon from './assets/icons/eye-off.svg';
import CodeIcon from './assets/icons/code.svg';
import MenuIcon from './assets/icons/menu.svg';

const md = new MarkdownIt();

// eslint-disable-next-line new-cap
const swapMachine = Machine({
  id: 'swap',
  initial: 'visible',
  states: {
    visible: {
      on: {
        TOGGLE: 'hidden',
      },
      initial: 'rendered',
      states: {
        rendered: {
          on: {
            SWAP: 'raw',
          },
        },
        raw: {
          on: {
            SWAP: 'rendered',
          },
        },
        memo: {
          type: 'history',
        },
      },
    },
    hidden: {
      on: {
        TOGGLE: 'visible.memo',
      },
    },
  },
});

const savedState = JSON.parse(localStorage.getItem('state'));
const savedData = localStorage.getItem('data');

const previousState = State.create(savedState || swapMachine.initialState);
const resolvedState = swapMachine.resolveState(previousState);

const persistData = debounce(data => {
  try {
    localStorage.setItem('data', data);
  } finally {
    // continue regardless of error
  }
}, 500);

export default {
  name: 'App',
  components: { EyeIcon, EyeOffIcon, CodeIcon, MenuIcon },
  data() {
    return {
      swapService: interpret(swapMachine),
      current: swapMachine.initialState,
      content: savedData
        ? savedData.trim()
        : '# Hello there!\n\n- Type some Markdown on the left\n- See HTML in the right\n- Magic\n\n![An orange jellyfish](https://i.picsum.photos/id/1069/400/250.jpg)',
    };
  },
  computed: {
    rendered() {
      return md.render(this.content);
    },
    raw() {
      return indent.html(this.rendered, {
        tabString: '  ',
      });
    },
  },
  methods: {
    send(event) {
      this.swapService.send(event);
    },
  },
  watch: {
    content(newContent) {
      persistData(newContent);
    },
  },
  created() {
    this.swapService
      .onTransition(state => {
        this.current = state;

        try {
          localStorage.setItem('state', JSON.stringify(this.current));
        } finally {
          // continue regardless of error
        }
      })
      .start(resolvedState);
  },
};
</script>

<style src="./assets/styles.css" />
