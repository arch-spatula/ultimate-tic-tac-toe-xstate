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
                  <li v-for="(innerCol, innerColIdx) in innerRow" :key="innerColIdx">
                    <button
                      :class="{
                        'square-btn': true,
                        'active-board':
                          snapshot.context.activeIdx.outerColIdx === outerColIdx &&
                          snapshot.context.activeIdx.outerRowIdx === outerRowIdx
                      }"
                      @click="move(innerColIdx, innerRowIdx, outerColIdx, outerRowIdx)"
                    >
                      <IconX v-if="innerCol === 'X'" color="#EF4444" />
                      <IconCircle v-if="innerCol === 'O'" color="#3B82F6" />
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
import { IconCircle, IconX } from '@tabler/icons-vue'

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
      activeIdx: { outerColIdx: number; outerRowIdx: number }
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
    activeIdx: { outerColIdx: -1, outerRowIdx: -1 }
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
      guard: ({ context, event }) => {
        // 시작하는 턴에 허용
        if (context.activeIdx.outerColIdx === -1 && context.activeIdx.outerRowIdx === -1)
          return true

        // 활성화된 보드만 허용
        if (
          context.activeIdx.outerColIdx !== event.value.outerColIdx ||
          context.activeIdx.outerRowIdx !== event.value.outerRowIdx
        )
          return false

        // empty가 아닐 때만 OX 표시 허용
        if (
          context.square[event.value.outerRowIdx][event.value.outerColIdx][event.value.innerRowIdx][
            event.value.innerColIdx
          ] !== 'empty'
        )
          return false

        return true
      },
      actions: assign({
        turn: ({ context }) => {
          if (context.turn === 'O') return 'X'
          else return 'O'
        },
        activeIdx: ({ event }) => {
          return { outerColIdx: event.value.innerColIdx, outerRowIdx: event.value.innerRowIdx }
        },
        square: ({ event, context }) => {
          if (
            context.square[event.value.outerRowIdx][event.value.outerColIdx][
              event.value.innerRowIdx
            ][event.value.innerColIdx] === 'empty'
          )
            context.square[event.value.outerRowIdx][event.value.outerColIdx][
              event.value.innerRowIdx
            ][event.value.innerColIdx] = context.turn
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
    background-color: #e5e7eb;
    cursor: pointer;
  }
  &:active {
    background-color: #d1d5db;
  }
}

.active-board {
  background-color: #f3f4f6;
}
</style>
