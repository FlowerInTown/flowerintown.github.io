<script setup lang="ts">
import { computed } from 'vue'

type KeyType = 'white' | 'black'

interface PianoKey {
  midi: number
  name: string
  type: KeyType
  whiteIndex: number // 第几个白键（0-based）
  octave: number
}

// 生成 88 键：A0(21) - C8(108)
function isBlack(midi: number) {
  const pc = midi % 12
  return [1, 3, 6, 8, 10].includes(pc)
}

function midiToName(midi: number) {
  const names = ['C', 'C#', 'D', 'D#', 'E', 'F', 'F#', 'G', 'G#', 'A', 'A#', 'B']
  const pc = midi % 12
  const oct = Math.floor(midi / 12) - 1
  return `${names[pc]}${oct}`
}

const allKeys: PianoKey[] = (() => {
  const res: PianoKey[] = []
  let whiteCount = 0
  for (let midi = 21; midi <= 108; midi++) {
    const black = isBlack(midi)
    const name = midiToName(midi)
    const oct = Math.floor(midi / 12) - 1
    if (!black) {
      res.push({ midi, name, type: 'white', whiteIndex: whiteCount, octave: oct })
      whiteCount++
    } else {
      res.push({ midi, name, type: 'black', whiteIndex: whiteCount - 1, octave: oct })
    }
  }
  return res
})()

const whiteKeys = computed(() => allKeys.filter(k => k.type === 'white'))
const blackKeys = computed(() => allKeys.filter(k => k.type === 'black'))

// 黑键横向偏移：贴近真琴
function blackOffsetForMidi(midi: number): number {
  const pc = midi % 12
  // 以内侧 C D E F G A B 七个白键为 0,1,2,3,4,5,6
  // C#/D# 稍偏右，F#/G#/A# 稍偏左
  if (pc === 1 || pc === 3) return 0.6   // C#, D#
  if (pc === 6 || pc === 8 || pc === 10) return 0.4 // F#, G#, A#
  return 0.5
}

// 组件向外暴露的事件：点击某键
const emit = defineEmits<{
  (e: 'key-down', midi: number): void
}>()
</script>

<template>
  <div class="piano-wrapper">
    <!-- 白键层 -->
    <div class="white-layer">
      <div
        v-for="key in whiteKeys"
        :key="'w-' + key.midi"
        class="white-key"
        @mousedown="emit('key-down', key.midi)"
      >
        <span class="white-label">{{ key.name }}</span>
      </div>
    </div>

    <!-- 黑键层 -->
    <div class="black-layer">
      <div
        v-for="key in blackKeys"
        :key="'b-' + key.midi"
        class="black-key"
        :style="{
          left: ((key.whiteIndex + blackOffsetForMidi(key.midi)) / whiteKeys.length) * 100 + '%'
        }"
        @mousedown.stop="emit('key-down', key.midi)"
      ></div>
    </div>
  </div>
</template>

<style scoped>
.piano-wrapper {
  position: relative;
  width: 100%;
  height: 140px;
  background: #e5e5e5;
  border-radius: 4px;
  overflow: hidden;
}

/* 白键层：用 flex 铺满 */
.white-layer {
  position: absolute;
  inset: 0;
  display: flex;
}

.white-key {
  flex: 1;
  position: relative;
  background: linear-gradient(to bottom, #ffffff, #f0f0f0);
  border-right: 1px solid #c5c5c5;
  box-shadow: inset 0 1px 0 rgba(255, 255, 255, 0.8);
  cursor: pointer;
}
.white-key:last-child {
  border-right: none;
}

.white-key:active {
  background: linear-gradient(to bottom, #f3f3f3, #e2e2e2);
}

.white-label {
  position: absolute;
  bottom: 2px;
  left: 4px;
  font-size: 9px;
  color: #444;
}

/* 黑键层：绝对覆盖在白键上方 */
.black-layer {
  position: absolute;
  inset: 0;
  pointer-events: none;
}

.black-key {
  position: absolute;
  pointer-events: auto;
  top: 0;
  width: calc(100% / 52 * 0.65);  /* 黑键略窄 */
  height: 60%;                    /* 高度约为白键 60% */
  transform: translateX(20%);
  background: linear-gradient(to bottom, #4a4a4a, #111111);
  border-radius: 0 0 4px 4px;
  border: 1px solid #000;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.7);
}

.black-key:active {
  background: linear-gradient(to bottom, #303030, #050505);
}
</style>