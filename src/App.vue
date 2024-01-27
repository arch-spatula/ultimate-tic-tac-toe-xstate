<template>
  <div class="container" v-if="snapshot.context.game === 'wait'">
    <div class="modal">
      <h3>보드 사이즈</h3>
      <div class="spinner">
        <!-- @todo: 비활성화 상태 포함하기 -->
        <button
          :class="{ 'number-btn': true, 'left-round': true }"
          @click="decrement"
          tabindex="-1"
        >
          <IconMinus color="#1F2937" />
        </button>
        <input
          class="size-input"
          type="number"
          :value="snapshot.context.size"
          @change="setSize"
          tabindex="-1"
          ref="inputRef"
        />
        <button class="number-btn right-round" @click="increment" tabindex="-1">
          <IconPlus color="#1F2937" />
        </button>
      </div>
      <button class="start-btn" @click="start">게임 시작</button>
    </div>
    <div class="overlay"></div>
  </div>
  <!-- @todo: 세로 가운데 정렬 -->
  <div class="board">
    <ul class="outer-row">
      <li v-for="(outerRow, outerRowIdx) in snapshot.context.square" :key="outerRowIdx">
        <ul class="outer-col">
          <li class="board-item" v-for="(outerCol, outerColIdx) in outerRow" :key="outerColIdx">
            <ul class="inner-row">
              <div v-if="snapshot.context.board[outerRowIdx][outerColIdx] !== 'empty'">
                <IconX
                  v-if="snapshot.context.board[outerRowIdx][outerColIdx] === 'X'"
                  color="#EF4444"
                />
                <IconCircle
                  v-if="snapshot.context.board[outerRowIdx][outerColIdx] === 'O'"
                  color="#3B82F6"
                />
              </div>
              <li v-else v-for="(innerRow, innerRowIdx) in outerCol" :key="innerRowIdx">
                <ul class="inner-col">
                  <li v-for="(innerCol, innerColIdx) in innerRow" :key="innerColIdx">
                    <button
                      :class="{
                        'square-btn': true,
                        'active-board':
                          (snapshot.context.activeIdx.outerColIdx === -1 &&
                            snapshot.context.activeIdx.outerRowIdx === -1) ||
                          (snapshot.context.activeIdx.outerColIdx === outerColIdx &&
                            snapshot.context.activeIdx.outerRowIdx === outerRowIdx)
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
import { IconCircle, IconX, IconPlus, IconMinus } from '@tabler/icons-vue'
import { ref } from 'vue'

type Player = 'O' | 'X'

type Mark = Player | 'empty'

const INIT_SQUARE = 3

const createEmptySquare = (size: number): Array<Mark> => {
  return Array.from({ length: size }, () => 'empty')
}

// 시작할 때 칸수 정의
// 칸수에 해당하는 중첩 배열 만들기
const gameMachine = createMachine({
  id: 'game',
  types: {} as {
    context: {
      game: 'wait' | 'play' | 'result'
      turn: Player
      square: Mark[][][][]
      board: Mark[][]
      size: number
      activeIdx: { outerColIdx: number; outerRowIdx: number }
      // 승부, o x draw
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
      | { type: 'setSize'; value: number }
      | { type: 'increment' }
      | { type: 'decrement' }
  },
  context: {
    game: 'wait',
    turn: 'O',
    square: Array.from({ length: INIT_SQUARE }, () => [
      ...Array.from({ length: INIT_SQUARE }, () => [
        ...Array.from({ length: INIT_SQUARE }, () => createEmptySquare(INIT_SQUARE))
      ])
    ]),
    board: Array.from({ length: INIT_SQUARE }, () => createEmptySquare(INIT_SQUARE)),
    size: INIT_SQUARE,
    activeIdx: { outerColIdx: -1, outerRowIdx: -1 }
  },
  on: {
    start: {
      guard: ({ context }) => {
        return context.game === 'wait'
      },
      actions: assign({ game: 'play' })
    },
    decision: {
      guard: ({ context }) => {
        return context.game === 'play'
      },
      actions: assign({ game: 'result' })
    },
    restart: {
      guard: ({ context }) => {
        return context.game === 'result'
      },
      actions: assign({ game: 'wait', turn: 'O', square: [] })
    },
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
        if (context.game !== 'play') return false
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
        activeIdx: ({ event, context }) => {
          // 데드락 풀기
          // empty가 아니면 전체로 초기화
          console.log(context.board[event.value.innerColIdx][event.value.innerColIdx])
          if (context.board[event.value.innerColIdx][event.value.innerColIdx] === 'empty')
            return { outerColIdx: event.value.innerColIdx, outerRowIdx: event.value.innerRowIdx }
          else return { outerColIdx: -1, outerRowIdx: -1 }
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
        },
        board: ({ event, context }) => {
          const currentBoard = context.square[event.value.outerRowIdx][event.value.outerColIdx]
          // 가로 검증
          currentBoard.forEach((row) => {
            const set = new Set(row)
            if (set.size === 1) {
              if (set.has('O')) {
                context.board[event.value.outerRowIdx][event.value.outerColIdx] = 'O'
                return context.board
              }
              if (set.has('X')) {
                context.board[event.value.outerRowIdx][event.value.outerColIdx] = 'X'
                return context.board
              }
            }
          })

          // 세로 검증
          for (let j = 0; j < context.size; j++) {
            const col = new Set<Mark>()
            for (let i = 0; i < context.size; i++) {
              col.add(currentBoard[i][j])
            }
            if (col.size === 1) {
              if (col.has('O')) {
                context.board[event.value.outerRowIdx][event.value.outerColIdx] = 'O'
                return context.board
              }
              if (col.has('X')) {
                context.board[event.value.outerRowIdx][event.value.outerColIdx] = 'X'
                return context.board
              }
            }
          }

          // 대각선 검증
          const leftToRight = new Set<Mark>()
          const rightToLeft = new Set<Mark>()
          for (let i = 0; i < context.size; i++) {
            leftToRight.add(currentBoard[i][i])
            rightToLeft.add(currentBoard[i][context.size - i - 1])
          }

          if (leftToRight.size === 1) {
            if (leftToRight.has('O')) {
              context.board[event.value.outerRowIdx][event.value.outerColIdx] = 'O'
              return context.board
            }
            if (leftToRight.has('X')) {
              context.board[event.value.outerRowIdx][event.value.outerColIdx] = 'X'
              return context.board
            }
          }

          if (rightToLeft.size === 1) {
            if (rightToLeft.has('O')) {
              context.board[event.value.outerRowIdx][event.value.outerColIdx] = 'O'
              return context.board
            }
            if (rightToLeft.has('X')) {
              context.board[event.value.outerRowIdx][event.value.outerColIdx] = 'X'
              return context.board
            }
          }

          return context.board
        }
      })
    },
    setSize: {
      guard: ({ context, event }) => {
        if (context.game === 'wait' && event.value >= INIT_SQUARE) return true
        else return false
      },
      actions: assign({
        size: ({ event }) => {
          return event.value
        },
        square: ({ event }) => {
          return Array.from({ length: event.value }, () => [
            ...Array.from({ length: event.value }, () => [
              ...Array.from({ length: event.value }, () => createEmptySquare(event.value))
            ])
          ])
        },
        board: ({ event }) =>
          Array.from({ length: event.value }, () => createEmptySquare(event.value))
      })
    },
    increment: {
      guard: ({ context }) => {
        if (context.game === 'wait') return true
        else return false
      },
      actions: assign({
        size: ({ context }) => {
          return context.size + 1
        },
        square: ({ context }) => {
          return Array.from({ length: context.size + 1 }, () => [
            ...Array.from({ length: context.size + 1 }, () => [
              ...Array.from({ length: context.size + 1 }, () => createEmptySquare(context.size + 1))
            ])
          ])
        },
        board: ({ context }) =>
          Array.from({ length: context.size + 1 }, () => createEmptySquare(context.size + 1))
      })
    },
    decrement: {
      guard: ({ context }) => {
        if (context.game === 'wait' && context.size > INIT_SQUARE) return true
        else return false
      },
      actions: assign({
        size: ({ context }) => {
          return context.size - 1
        },
        square: ({ context }) => {
          return Array.from({ length: context.size - 1 }, () => [
            ...Array.from({ length: context.size - 1 }, () => [
              ...Array.from({ length: context.size - 1 }, () => createEmptySquare(context.size - 1))
            ])
          ])
        },
        board: ({ context }) =>
          Array.from({ length: context.size - 1 }, () => createEmptySquare(context.size - 1))
      })
    }
  }
})

const { send, snapshot } = useMachine(gameMachine)

const inputRef = ref<HTMLInputElement | null>(null)

function start() {
  send({ type: 'start' })
}

function decision() {
  send({ type: 'decision' })
}

function restart() {
  send({ type: 'restart' })
}

function setSize(e: Event) {
  const size = parseInt((e.target as HTMLInputElement).value)
  if (size < 3) send({ type: 'setSize', value: 3 })
  else send({ type: 'setSize', value: size })
}

function increment() {
  send({ type: 'increment' })
  if (inputRef.value) inputRef.value.focus()
}

function decrement() {
  send({ type: 'decrement' })
  if (snapshot.value.context.size > 3 && inputRef.value) inputRef.value.focus()
}

function move(innerColId: number, innerRowIdx: number, outerColIdx: number, outerRowIdx: number) {
  send({ type: 'move', value: { innerColIdx: innerColId, innerRowIdx, outerColIdx, outerRowIdx } })
}

const squareColor50: Record<Player, string> = { X: '#FEF2F2', O: '#EFF6FF' } as const
const squareColor100: Record<Player, string> = { X: '#FEE2E2', O: '#DBEAFE' } as const
</script>

<style scoped lang="scss">
.board {
  min-height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
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

  .board-item {
    width: 100%;
    display: flex;
    justify-content: center;
    align-items: center;
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
}

.square-btn {
  all: unset;
  width: 3.75rem;
  height: 3.75rem;
  background-color: #f3f4f6;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 0.5rem;
  &:hover {
    background-color: v-bind('squareColor50[snapshot.context.turn]');
    cursor: pointer;
  }
  &:active {
    background-color: v-bind('squareColor100[snapshot.context.turn]');
  }
}

.active-board {
  background-color: #f9fafb;
}

.container {
  position: absolute;
  display: flex;
  justify-content: center;
  align-items: center;
  .modal {
    position: absolute;
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    gap: 2rem;
    width: 30rem;
    height: 30rem;
    background-color: #ffffff;
    border-radius: 1.5rem;

    .start-btn {
      height: 3.75rem;
      padding: 0.5rem 2rem;
      border-radius: 0.5rem;
      font-size: larger;
      background-color: #f3f4f6;
      &:hover {
        background-color: #e5e7eb;
        cursor: pointer;
      }
      &:active {
        background-color: #d1d5db;
      }
    }

    .spinner {
      display: flex;
      flex-direction: row;
      .number-btn {
        height: 3.75rem;
        width: 3.75rem;
        display: flex;
        align-items: center;
        justify-content: center;
        background-color: #f3f4f6;
        &:hover {
          /* snapshot.value.context.turn */
          /* background-color: #EFF6FF; bg-blue-50 */
          /* background-color: #FEF2F2; bg-red-50 */
          background-color: #e5e7eb;
          cursor: pointer;
        }
        &:active {
          background-color: #d1d5db;
        }
      }

      .size-input {
        height: 3.75rem;
        text-align: center;
        font-size: large;
      }

      .right-round {
        border-radius: 0 0.5rem 0.5rem 0;
      }

      .left-round {
        border-radius: 0.5rem 0 0 0.5rem;
      }

      input::-webkit-outer-spin-button,
      input::-webkit-inner-spin-button {
        /* display: none; <- Crashes Chrome on hover */
        -webkit-appearance: none;
        margin: 0; /* <-- Apparently some margin are still there even though it's hidden */
        outline: none;
      }
      input[type='number'] {
        /* Firefox */
        appearance: textfield;
        /* -moz-appearance: textfield; */
        outline: none;
      }
    }
  }

  .overlay {
    background-color: #00000004;
    width: 100vw;
    height: 100vh;
  }
}
</style>
