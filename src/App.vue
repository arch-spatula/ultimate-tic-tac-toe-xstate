<template>
  <div>{{ snapshot.context.game }}</div>
  <button @click="start">start</button>
  <button @click="decision">decision</button>
  <button @click="restart">restart</button>
</template>

<script setup lang="ts">
import { useCssModule } from 'vue'
import { createMachine, assign } from 'xstate'
import { useMachine } from '@xstate/vue'

// 대기(wait) -시작-> 플레이(play) -판정-> 승부(result) -재시작-> 대기
// O -> X -> O
//
const gameMachine = createMachine({
  id: 'game',
  types: {} as {
    context: {
      game: 'wait' | 'play' | 'result'
    }
    events: { type: 'start' } | { type: 'decision' } | { type: 'restart' }
  },
  context: { game: 'wait' },
  on: {
    start: { actions: assign({ game: 'play' }) },
    decision: { actions: assign({ game: 'result' }) },
    restart: { actions: assign({ game: 'wait' }) }
  }
})

const { send, snapshot } = useMachine(gameMachine)

function start() {
  send({ type: 'start' })
}

function decision() {
  send({ type: 'decision' })
}

function restart() {
  send({ type: 'restart' })
}
</script>

<style scoped>
header {
  line-height: 1.5;
  max-height: 100vh;
}

.logo {
  display: block;
  margin: 0 auto 2rem;
}

nav {
  width: 100%;
  font-size: 12px;
  text-align: center;
  margin-top: 2rem;
}

nav a.router-link-exact-active {
  color: var(--color-text);
}

nav a.router-link-exact-active:hover {
  background-color: transparent;
}

nav a {
  display: inline-block;
  padding: 0 1rem;
  border-left: 1px solid var(--color-border);
}

nav a:first-of-type {
  border: 0;
}

@media (min-width: 1024px) {
  header {
    display: flex;
    place-items: center;
    padding-right: calc(var(--section-gap) / 2);
  }

  .logo {
    margin: 0 2rem 0 0;
  }

  header .wrapper {
    display: flex;
    place-items: flex-start;
    flex-wrap: wrap;
  }

  nav {
    text-align: left;
    margin-left: -1rem;
    font-size: 1rem;

    padding: 1rem 0;
    margin-top: 1rem;
  }
}
</style>
