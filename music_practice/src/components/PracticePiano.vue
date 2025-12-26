<script setup lang="ts">
import {
  ref,
  computed,
  onMounted,
  onBeforeUnmount,
  watch,
  nextTick,
} from 'vue'
import * as Tone from 'tone'
import { OpenSheetMusicDisplay } from 'opensheetmusicdisplay'

type Mode = 'C_MAJOR' | 'G_MAJOR' | 'A_MINOR'
type Difficulty = 1 | 2 | 3
type Staff = 'treble' | 'bass'

interface NoteEvent {
  time: number
  duration: number
  midi: number
  staff: Staff
  isChord: boolean
}

// 顶部控制
const modes: { label: string; value: Mode }[] = [
  { label: 'C 大调', value: 'C_MAJOR' },
  { label: 'G 大调', value: 'G_MAJOR' },
  { label: 'A 小调', value: 'A_MINOR' }
]
const meters = ['4/4'] as const
const tempos = [60, 80, 100, 120, 140]

const selectedMode = ref<Mode>('C_MAJOR')
const selectedMeter = ref<(typeof meters)[number]>('4/4')
const bpm = ref(100)
const difficulty = ref<Difficulty>(1)
const showPianoAuto = ref(true)

// 乐句：固定 4 小节一行
const phraseLengthBars = 4
const phraseDuration = computed(() => {
  const [beatsPerBarStr] = selectedMeter.value.split('/')
  const beatsPerBar = Number(beatsPerBarStr)
  const secondsPerBeat = 60 / bpm.value
  return beatsPerBar * secondsPerBeat * phraseLengthBars
})

const noteEvents = ref<NoteEvent[]>([])
const phraseStart = ref(0)
const isPlaying = ref(false)

// Tone.js Sampler
let pianoSampler: Tone.Sampler | null = null
const samplerLoaded = ref(false)

async function ensureAudio() {
  await Tone.start()
  if (!pianoSampler) {
    pianoSampler = new Tone.Sampler(
      {
        A0: 'A0.mp3',
        C1: 'C1.mp3',
        'D#1': 'Ds1.mp3',
        'F#1': 'Fs1.mp3',
        A1: 'A1.mp3',
        C2: 'C2.mp3',
        'D#2': 'Ds2.mp3',
        'F#2': 'Fs2.mp3',
        A2: 'A2.mp3',
        C3: 'C3.mp3',
        'D#3': 'Ds3.mp3',
        'F#3': 'Fs3.mp3',
        A3: 'A3.mp3',
        C4: 'C4.mp3',
        'D#4': 'Ds4.mp3',
        'F#4': 'Fs4.mp3',
        A4: 'A4.mp3',
        C5: 'C5.mp3',
        'D#5': 'Ds5.mp3',
        'F#5': 'Fs5.mp3',
        A5: 'A5.mp3',
        C6: 'C6.mp3',
        'D#6': 'Ds6.mp3',
        'F#6': 'Fs6.mp3',
        A6: 'A6.mp3',
        C7: 'C7.mp3',
        'D#7': 'Ds7.mp3',
        'F#7': 'Fs7.mp3',
        A7: 'A7.mp3',
        C8: 'C8.mp3'
      },
      {
        baseUrl: 'https://tonejs.github.io/audio/salamander/',
        onload: () => {
          samplerLoaded.value = true
        }
      }
    ).toDestination()
    pianoSampler.volume.value = -6
  }
  Tone.getTransport().bpm.value = bpm.value
}

async function playMidiOnce(midi: number) {
  await ensureAudio()
  if (!pianoSampler || !samplerLoaded.value) return
  pianoSampler!.triggerAttackRelease(
    Tone.Frequency(midi, 'midi').toNote(),
    0.6,
    Tone.now()
  )
}

// 88 键键盘
interface PianoKeyDef {
  midi: number
  name: string
  isBlack: boolean
  whiteIndex: number
}
function midiToName(midi: number) {
  const names = ['C', 'C#', 'D', 'D#', 'E', 'F', 'F#', 'G', 'G#', 'A', 'A#', 'B']
  const pc = midi % 12
  const oct = Math.floor(midi / 12) - 1
  return `${names[pc]}${oct}`
}
function isBlack(midi: number) {
  const pc = midi % 12
  return [1,3,6,8,10].includes(pc)
}
const PIANO_KEYS: PianoKeyDef[] = (() => {
  const res: PianoKeyDef[] = []
  let whiteCount = 0
  for (let midi = 21; midi <= 108; midi++) {
    const black = isBlack(midi)
    const name = midiToName(midi)
    if (!black) {
      res.push({ midi, name, isBlack: false, whiteIndex: whiteCount })
      whiteCount++
    } else {
      res.push({ midi, name, isBlack: true, whiteIndex: whiteCount - 1 })
    }
  }
  return res
})()
const whiteKeys = PIANO_KEYS.filter(k => !k.isBlack)
const blackKeys = PIANO_KEYS.filter(k => k.isBlack)

function blackOffsetForMidi(midi: number): number {
  const pc = midi % 12
  if (pc === 1 || pc === 3) return 0.6
  if (pc === 6 || pc === 8 || pc === 10) return 0.4
  return 0.5
}

// 自动高亮
const activeMidiSet = ref<Set<number>>(new Set())
function updateActiveNotes() {
  if (!isPlaying.value) {
    activeMidiSet.value = new Set()
  } else {
    const elapsed = (Date.now() - phraseStart.value) / 1000
    const set = new Set<number>()
    noteEvents.value.forEach((ev) => {
      if (elapsed >= ev.time && elapsed <= ev.time + ev.duration) {
        set.add(ev.midi)
      }
    })
    activeMidiSet.value = set
  }
  requestAnimationFrame(updateActiveNotes)
}

// 乐句生成：跟之前一样
function getScaleMidiRange(mode: Mode, staff: Staff): { min: number; max: number } {
  if (staff === 'treble') return { min: 60, max: 84 }
  return { min: 36, max: 60 }
}
function randomInt(min: number, max: number) {
  return Math.floor(Math.random() * (max - min + 1)) + min
}
function snapToTriad(midi: number, mode: Mode) {
  const triads = [48, 53, 55]
  let closest = triads[0]
  let diff = Infinity
  for (const t of triads) {
    const d = Math.abs(t - midi)
    if (d < diff) {
      diff = d
      closest = t
    }
  }
  return closest
}
function buildChord(root: number, diff: Difficulty): number[] {
  const third = root + 4
  const fifth = root + 7
  const chord = [root, third, fifth]
  if (diff >= 2) chord.push(root - 12)
  if (diff >= 3) chord.push(root + 11)
  return chord
}

function generatePhrase() {
  const events: NoteEvent[] = []
  const [beatsPerBarStr] = selectedMeter.value.split('/')
  const beatsPerBar = Number(beatsPerBarStr)
  const secondsPerBeat = 60 / bpm.value
  const totalBeats = beatsPerBar * phraseLengthBars

  for (let beatIndex = 0; beatIndex < totalBeats; beatIndex++) {
    const beatInBar = beatIndex % beatsPerBar
    const t = beatIndex * secondsPerBeat

    const tr = getScaleMidiRange(selectedMode.value, 'treble')
    const trebleMidi = randomInt(tr.min, tr.max)
    events.push({
      time: t,
      duration: secondsPerBeat * 0.9,
      midi: trebleMidi,
      staff: 'treble',
      isChord: false
    })

    if (beatInBar === 0) {
      const br = getScaleMidiRange(selectedMode.value, 'bass')
      const root = snapToTriad(randomInt(br.min, br.max), selectedMode.value)
      const chordNotes = buildChord(root, difficulty.value)
      chordNotes.forEach((m) => {
        events.push({
          time: t,
          duration: secondsPerBeat * beatsPerBar,
          midi: m,
          staff: 'bass',
          isChord: true
        })
      })
    }
  }

  noteEvents.value = events
  nextTick(renderOsmdFromPhrase)
}

// 播放 & 循环
let playbackTimers: number[] = []
function clearPlayback() {
  playbackTimers.forEach(id => clearTimeout(id))
  playbackTimers = []
}
async function startPlayback() {
  await ensureAudio()
  if (!pianoSampler || !samplerLoaded.value) return

  clearPlayback()
  phraseStart.value = Date.now()
  isPlaying.value = true

  noteEvents.value.forEach((ev) => {
    const delayMs = ev.time * 1000
    const id = window.setTimeout(() => {
      if (!pianoSampler || !samplerLoaded.value || !isPlaying.value) return
      pianoSampler!.triggerAttackRelease(
        Tone.Frequency(ev.midi, 'midi').toNote(),
        ev.duration,
        Tone.now()
      )
    }, delayMs)
    playbackTimers.push(id)
  })

  const totalMs = phraseDuration.value * 1000
  const loopId = window.setTimeout(() => {
    if (!isPlaying.value) return
    generatePhrase()
    startPlayback()
  }, totalMs + 50)
  playbackTimers.push(loopId)
}
function stopPlayback() {
  isPlaying.value = false
  clearPlayback()
}

// 进度条
const progress = ref(0)
let progressRAF: number | null = null
function updateProgress() {
  if (!isPlaying.value) {
    progress.value = 0
  } else {
    const elapsed = (Date.now() - phraseStart.value) / 1000
    let p = elapsed / phraseDuration.value
    if (p > 1) p = 1
    progress.value = p
  }
  progressRAF = requestAnimationFrame(updateProgress)
}
watch(isPlaying, (p) => {
  if (p) {
    if (progressRAF === null) progressRAF = requestAnimationFrame(updateProgress)
  } else if (progressRAF !== null) {
    cancelAnimationFrame(progressRAF)
    progressRAF = null
  }
})
watch([bpm, selectedMeter, selectedMode, difficulty], () => {
  generatePhrase()
})

// OSMD 双排钢琴模板
const PIANO_TEMPLATE_XML = `<?xml version="1.0" encoding="UTF-8"?>
<score-partwise version="3.1"
  xmlns="http://www.musicxml.org/ns/musicxml">
  <part-list>
    <score-part id="P1">
      <part-name>Piano</part-name>
    </score-part>
  </part-list>
  <part id="P1">
    <measure number="0">
      <attributes>
        <divisions>4</divisions>
      </attributes>
      <direction placement="above">
        <direction-type>
          <metronome>
            <beat-unit>quarter</beat-unit>
            <per-minute>{{TEMPO}}</per-minute>
          </metronome>
        </direction-type>
      </direction>
    </measure>

    <measure number="1">
      <attributes>
        <divisions>4</divisions>
        <key><fifths>0</fifths></key>
        <time><beats>4</beats><beat-type>4</beat-type></time>
        <staves>2</staves>
        <clef number="1"><sign>G</sign><line>2</line></clef>
        <clef number="2"><sign>F</sign><line>4</line></clef>
      </attributes>
      {{T_M1_B1}}{{T_M1_B2}}{{T_M1_B3}}{{T_M1_B4}}
      {{B_M1_GROUP}}
    </measure>

    <measure number="2">
      {{T_M2_B1}}{{T_M2_B2}}{{T_M2_B3}}{{T_M2_B4}}
      {{B_M2_GROUP}}
    </measure>

    <measure number="3">
      {{T_M3_B1}}{{T_M3_B2}}{{T_M3_B3}}{{T_M3_B4}}
      {{B_M3_GROUP}}
    </measure>

    <measure number="4">
      {{T_M4_B1}}{{T_M4_B2}}{{T_M4_B3}}{{T_M4_B4}}
      {{B_M4_GROUP}}
    </measure>
  </part>
</score-partwise>`

// midi -> MusicXML pitch
function midiToXmlPitch(midi: number) {
  const names = ['C','C#','D','D#','E','F','F#','G','G#','A','A#','B']
  const pc = midi % 12
  const oct = Math.floor(midi / 12) - 1
  const name = names[pc]
  const step = name[0]
  const alter = name.includes('#') ? 1 : 0
  return { step, alter, octave: oct }
}

// 根据 noteEvents 填充模板
function buildMusicXmlFromPhrase(): string {
  const secPerBeat = 60 / bpm.value
  const beatsPerBar = 4
  const divisions = 4

  const trebleEvents = [...noteEvents.value]
    .filter(n => n.staff === 'treble')
    .sort((a, b) => a.time - b.time)

  const bassChordStarts = [...noteEvents.value]
    .filter(n => n.staff === 'bass')
    .sort((a, b) => a.time - b.time)

  function buildTrebleBeatNote(bar: number, beat: number): string {
    const measureStart = (bar - 1) * beatsPerBar * secPerBeat
    const bStart = measureStart + (beat - 1) * secPerBeat
    const bEnd = bStart + secPerBeat
    const ev = trebleEvents.find(e => e.time >= bStart && e.time < bEnd)
    if (!ev) {
      return `
        <note>
          <rest/>
          <duration>${divisions}</duration>
          <voice>1</voice>
          <type>quarter</type>
          <staff>1</staff>
        </note>`
    }
    const { step, alter, octave } = midiToXmlPitch(ev.midi)
    return `
      <note>
        <pitch>
          <step>${step}</step>
          ${alter ? `<alter>${alter}</alter>` : ''}
          <octave>${octave}</octave>
        </pitch>
        <duration>${divisions}</duration>
        <voice>1</voice>
        <type>quarter</type>
        <staff>1</staff>
      </note>`
  }

  function buildBassGroup(bar: number): string {
    const measureStart = (bar - 1) * beatsPerBar * secPerBeat
    const chordEvents = bassChordStarts.filter(e => e.time === measureStart)
    if (!chordEvents.length) {
      let restXml = ''
      for (let i = 0; i < 4; i++) {
        restXml += `
          <note>
            <rest/>
            <duration>${divisions / 2}</duration>
            <voice>1</voice>
            <type>eighth</type>
            <staff>2</staff>
          </note>`
      }
      return restXml
    }
    const pitches = chordEvents.map(e => e.midi).sort((a,b)=>a-b)
    const pattern: number[] = []
    pattern.push(pitches[0])
    if (pitches[1]) pattern.push(pitches[1])
    if (pitches[2]) pattern.push(pitches[2])
    pattern.push(pitches[0])

    let xml = ''
    pattern.forEach((m, idx) => {
      const { step, alter, octave } = midiToXmlPitch(m)
      const beamTag =
        idx === 0 ? '<beam number="1">begin</beam>' :
        idx === pattern.length - 1 ? '<beam number="1">end</beam>' :
        '<beam number="1">continue</beam>'
      xml += `
        <note>
          <pitch>
            <step>${step}</step>
            ${alter ? `<alter>${alter}</alter>` : ''}
            <octave>${octave}</octave>
          </pitch>
          <duration>${divisions / 2}</duration>
          <voice>1</voice>
          <type>eighth</type>
          ${beamTag}
          <staff>2</staff>
        </note>`
    })
    return xml
  }

  let xml = PIANO_TEMPLATE_XML
    .replace('{{TEMPO}}', String(bpm.value))

    .replace('{{T_M1_B1}}', buildTrebleBeatNote(1,1))
    .replace('{{T_M1_B2}}', buildTrebleBeatNote(1,2))
    .replace('{{T_M1_B3}}', buildTrebleBeatNote(1,3))
    .replace('{{T_M1_B4}}', buildTrebleBeatNote(1,4))
    .replace('{{B_M1_GROUP}}', buildBassGroup(1))

    .replace('{{T_M2_B1}}', buildTrebleBeatNote(2,1))
    .replace('{{T_M2_B2}}', buildTrebleBeatNote(2,2))
    .replace('{{T_M2_B3}}', buildTrebleBeatNote(2,3))
    .replace('{{T_M2_B4}}', buildTrebleBeatNote(2,4))
    .replace('{{B_M2_GROUP}}', buildBassGroup(2))

    .replace('{{T_M3_B1}}', buildTrebleBeatNote(3,1))
    .replace('{{T_M3_B2}}', buildTrebleBeatNote(3,2))
    .replace('{{T_M3_B3}}', buildTrebleBeatNote(3,3))
    .replace('{{T_M3_B4}}', buildTrebleBeatNote(3,4))
    .replace('{{B_M3_GROUP}}', buildBassGroup(3))

    .replace('{{T_M4_B1}}', buildTrebleBeatNote(4,1))
    .replace('{{T_M4_B2}}', buildTrebleBeatNote(4,2))
    .replace('{{T_M4_B3}}', buildTrebleBeatNote(4,3))
    .replace('{{T_M4_B4}}', buildTrebleBeatNote(4,4))
    .replace('{{B_M4_GROUP}}', buildBassGroup(4))

  return xml
}

// OSMD 实例
const scoreContainerRef = ref<HTMLDivElement | null>(null)
let osmd: OpenSheetMusicDisplay | null = null

async function renderOsmdFromPhrase() {
  const container = scoreContainerRef.value
  if (!container) return

  if (!osmd) {
    osmd = new OpenSheetMusicDisplay(container, {
      autoResize: true,
      drawTitle: false,
      drawComposer: false,
      drawPartNames: false,
    })
  }

  const xml = buildMusicXmlFromPhrase()
  await osmd.load(xml)
  osmd.render()
}

// 生命周期
onMounted(() => {
  ensureAudio().catch(() => {})
  generatePhrase()
  updateActiveNotes()
})
onBeforeUnmount(() => {
  stopPlayback()
  Tone.Transport.stop()
  Tone.Transport.cancel(0)
})
</script>

<template>
  <div class="app-root">
    <header class="top-bar">
      <div class="logo">
        <span class="dot"></span>
        <span class="text">Piano Flow</span>
      </div>
      <div class="controls">
        <div class="control-group">
          <label>调式</label>
          <select v-model="selectedMode">
            <option v-for="m in modes" :key="m.value" :value="m.value">
              {{ m.label }}
            </option>
          </select>
        </div>
        <div class="control-group">
          <label>节拍</label>
          <select v-model="selectedMeter">
            <option v-for="m in meters" :key="m" :value="m">{{ m }}</option>
          </select>
        </div>
        <div class="control-group">
          <label>BPM</label>
          <select v-model="bpm">
            <option v-for="t in tempos" :key="t" :value="t">{{ t }}</option>
          </select>
        </div>
        <div class="control-group">
          <label>难度</label>
          <select v-model="difficulty">
            <option :value="1">1</option>
            <option :value="2">2</option>
            <option :value="3">3</option>
          </select>
        </div>
        <div class="control-group toggle">
          <label>
            <input type="checkbox" v-model="showPianoAuto" />
            自动显示琴键
          </label>
        </div>
        <div class="control-group">
          <button
            class="btn primary"
            @click="startPlayback"
            :disabled="isPlaying || !samplerLoaded"
          >
            ▶ 播放
          </button>
          <button class="btn ghost" @click="stopPlayback" :disabled="!isPlaying">
            ■ 停止
          </button>
        </div>
      </div>
    </header>

    <main class="main-layout">
      <section class="score-panel">
        <div class="staff-card">
          <div ref="scoreContainerRef" class="score-container"></div>
        </div>
      </section>

      <section class="piano-panel">
        <div class="piano-wrapper">
          <div class="piano">
            <div class="white-key-row">
              <div
                v-for="key in whiteKeys"
                :key="'w-' + key.midi"
                class="white-key"
                :class="{ active: showPianoAuto && activeMidiSet.has(key.midi) }"
                @mousedown="playMidiOnce(key.midi)"
              >
                <span class="key-label">{{ key.name }}</span>
              </div>
            </div>
            <div class="black-key-row">
              <div
                v-for="b in blackKeys"
                :key="'b-' + b.midi"
                class="black-key"
                :class="{ active: showPianoAuto && activeMidiSet.has(b.midi) }"
                :style="{
                  left: ((b.whiteIndex + blackOffsetForMidi(b.midi)) / whiteKeys.length) * 100 + '%'
                }"
                @mousedown.stop="playMidiOnce(b.midi)"
              ></div>
            </div>
          </div>
        </div>
      </section>
    </main>

    <footer class="footer">
      <div class="status">
        <span class="status-dot" :class="samplerLoaded ? 'ok' : 'loading'"></span>
        <span v-if="samplerLoaded">钢琴音色已加载</span>
        <span v-else>正在加载钢琴音色…</span>
      </div>
    </footer>
  </div>
</template>

<style scoped>
.app-root {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  background: radial-gradient(circle at top, #e3f2fd 0, #f5fbff 40%, #ffffff 100%);
  color: #1f2933;
  box-sizing: border-box;
  overflow: hidden;
}
.top-bar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 8px 20px;
  background: rgba(255, 255, 255, 0.9);
  border-bottom: 1px solid rgba(15, 23, 42, 0.06);
}
.logo {
  display: flex;
  align-items: center;
  gap: 8px;
  font-weight: 700;
  letter-spacing: 0.08em;
  font-size: 14px;
  text-transform: uppercase;
  color: #1d4ed8;
}
.logo .dot {
  width: 10px;
  height: 10px;
  border-radius: 999px;
  background: #22c55e;
  box-shadow: 0 0 8px rgba(34, 197, 94, 0.7);
}
.controls {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  gap: 8px 14px;
}
.control-group {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 12px;
}
.control-group label {
  color: #4b5563;
}
.control-group select {
  border-radius: 999px;
  border: 1px solid rgba(148, 163, 184, 0.7);
  padding: 4px 10px;
  font-size: 12px;
  background: rgba(248, 250, 252, 0.9);
}
.btn {
  border-radius: 999px;
  border: none;
  padding: 4px 12px;
  font-size: 12px;
  cursor: pointer;
}
.btn.primary {
  background: linear-gradient(135deg, #2563eb, #22c55e);
  color: #f9fafb;
}
.btn.ghost {
  background: rgba(255, 255, 255, 0.7);
  color: #1f2937;
  border: 1px solid rgba(148, 163, 184, 0.7);
}
.btn:disabled {
  opacity: 0.5;
  cursor: default;
}

.main-layout {
  flex: 1;
  display: flex;
  flex-direction: column;
  padding: 8px 20px 4px;
  gap: 8px;
  box-sizing: border-box;
  overflow: hidden;
}
.score-panel {
  flex: 1;
  min-height: 0;
}
.piano-panel {
  height: 200px;
}

.staff-card {
  height: 100%;
  background: #ffffff;
  border-radius: 12px;
  padding: 10px 10px;
  box-shadow: 0 16px 32px rgba(15, 23, 42, 0.08);
  border: 1px solid rgba(148, 163, 184, 0.35);
  box-sizing: border-box;
}
.score-container {
  width: 100%;
  height: 100%;
}

.piano-wrapper {
  position: relative;
  height: 100%;
  border-radius: 12px;
  padding: 10px;
  background: radial-gradient(circle at 10% 20%, #e5e7eb 0, #f9fafb 40%, #ffffff 100%);
  box-shadow: 0 16px 32px rgba(15, 23, 42, 0.12);
  border: 1px solid rgba(148, 163, 184, 0.4);
  overflow: hidden;
}
.piano {
  position: relative;
  height: 100%;
}
.white-key-row {
  position: absolute;
  inset: 0;
  display: flex;
}
.white-key {
  flex: 1;
  position: relative;
  background: linear-gradient(to bottom, #ffffff, #f1f1f1);
  border-right: 1px solid #c5c5c5;
  box-shadow: inset 0 1px 0 rgba(255, 255, 255, 0.9);
  cursor: pointer;
}
.white-key:last-child {
  border-right: none;
}
.white-key.active {
  background: linear-gradient(to bottom, #fef9c3, #fde68a);
  border-color: #facc15;
}
.key-label {
  position: absolute;
  bottom: 2px;
  left: 4px;
  font-size: 9px;
  color: #4b5563;
}
.black-key-row {
  position: absolute;
  inset: 0;
  pointer-events: none;
}
.black-key {
  position: absolute;
  pointer-events: auto;
  top: 0;
  width: calc(100% / 52 * 0.65);
  height: 60%;
  transform: translateX(-50%);
  background: linear-gradient(to bottom, #4a4a4a, #111111);
  border-radius: 0 0 4px 4px;
  border: 1px solid #000;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.8);
}
.black-key.active {
  background: linear-gradient(to bottom, #f97316, #ea580c);
}

.footer {
  padding: 4px 20px 6px;
  font-size: 11px;
  color: #6b7280;
  display: flex;
  justify-content: flex-end;
}
.status {
  display: flex;
  align-items: center;
  gap: 6px;
}
.status-dot {
  width: 8px;
  height: 8px;
  border-radius: 999px;
  background: #9ca3af;
}
.status-dot.loading {
  background: #f59e0b;
}
.status-dot.ok {
  background: #22c55e;
}
</style>