<template>
  <div>현재 게임 상태: {{ snapshot.context.game }}</div>
  <div>
    <button @click="start">start</button>
    <button @click="decision">decision</button>
    <button @click="restart">restart</button>
  </div>
  <div>현재 플레이어 턴: {{ snapshot.context.turn }}</div>
  <div>
    <button @click="toggle">toggle</button>
    <div>{{ snapshot.context.activeIdx }}</div>
  </div>
  <div class="board">
    <div>보드</div>
    <ul class="outer-row">
      <li v-for="(outerRow, outerRowIdx) in snapshot.context.square" :key="outerRowIdx">
        <ul class="outer-col">
          <li v-for="(outerCol, outerColIdx) in outerRow" :key="outerColIdx">
            <ul class="inner-row">
              <li v-for="(innerRow, innerRowIdx) in outerCol" :key="innerRowIdx">
                <ul class="inner-col">
                  <li v-for="(innerCol, innerColId) in innerRow" :key="innerColId">
                    <button
                      class="square-btn"
                      @click="move(innerColId, innerRowIdx, outerColIdx, outerRowIdx)"
                    >
                      <p>{{ innerCol }}</p>
                    </button>
                  </li>
                </ul>
              </li>
            </ul>
          </li>
        </ul>
      </li>
    </ul>
  </div>
</template>

<script setup lang="ts">
import { createMachine, assign } from 'xstate'
import { useMachine } from '@xstate/vue'

const INIT_SQUARE = 3

const createEmptySquare = (size: number): Array<'O' | 'X' | 'empty'> => {
  return Array.from({ length: size }, () => 'empty')
}

// 대기(wait) -시작-> 플레이(play) -판정-> 승부(result) -재시작-> 대기
// O -> X -> O
// 시작할 때 칸수 정의
// 칸수에 해당하는 중첩 배열 만들기
const gameMachine = createMachine({
  id: 'game',
  types: {} as {
    context: {
      game: 'wait' | 'play' | 'result'
      turn: 'O' | 'X'
      square: ('O' | 'X' | 'empty')[][][][]
      size: number
      activeIdx: number
    }
    events:
      | { type: 'start' }
      | { type: 'decision' }
      | { type: 'restart' }
      | { type: 'toggle' }
      | {
          type: 'move'
          value: {
            innerColIdx: number
            innerRowIdx: number
            outerColIdx: number
            outerRowIdx: number
          }
        }
  },
  context: {
    game: 'wait',
    turn: 'O',
    square: Array.from({ length: INIT_SQUARE }, () => [
      ...Array.from({ length: INIT_SQUARE }, () => [
        ...Array.from({ length: INIT_SQUARE }, () => createEmptySquare(INIT_SQUARE))
      ])
    ]),
    size: 3,
    activeIdx: -1
  },
  on: {
    start: { actions: assign({ game: 'play' }) },
    decision: { actions: assign({ game: 'result' }) },
    restart: { actions: assign({ game: 'wait', turn: 'O', square: [] }) },
    toggle: {
      actions: assign({
        turn: ({ context }) => {
          if (context.turn === 'O') return 'X'
          else return 'O'
        }
      })
    },
    move: {
      actions: assign({
        turn: ({ context }) => {
          if (context.turn === 'O') return 'X'
          else return 'O'
        },
        activeIdx: ({ event }) => {
          return event.value.innerColIdx + event.value.innerRowIdx * 3
        },
        square: ({ event, context }) => {
          context.square[event.value.outerRowIdx][event.value.outerColIdx][event.value.innerRowIdx][
            event.value.innerColIdx
          ] = context.turn
          return context.square
        }
      })
    }
    // setSize
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

function toggle() {
  send({ type: 'toggle' })
}

function move(innerColId: number, innerRowIdx: number, outerColIdx: number, outerRowIdx: number) {
  send({ type: 'move', value: { innerColIdx: innerColId, innerRowIdx, outerColIdx, outerRowIdx } })
}
</script>

<style scoped lang="scss">
.outer-col {
  display: flex;
  gap: 2rem;
}

.outer-row {
  display: flex;
  gap: 1.75rem;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}

.inner-col {
  display: flex;
  gap: 1rem;
}

.inner-row {
  display: flex;
  gap: 0.75rem;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}

.square-btn {
  all: unset;
  width: 3.75rem;
  height: 3.75rem;
  background-color: #f9fafb;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 0.5rem;
  &:hover {
    background-color: #f3f4f6;
    cursor: pointer;
  }
  &:active {
    background-color: #e5e7eb;
  }
}
</style>
