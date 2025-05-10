<script setup>
import data from "@/data.json"
import { computed, ref } from "vue"
import OpenAI from "openai"
import { zodTextFormat } from "openai/helpers/zod"
import { z } from "zod"

const openai = new OpenAI({
  apiKey: import.meta.env.VITE_OPENAI_KEY,
  dangerouslyAllowBrowser: true
})

const Quiz = z.object({
  questions: z.array(z.object({
    text: z.string(),
    options: z.array(z.string()),
    answer: z.string()
  })),
})

const prices = [
  "500",
  "1 000",
  "2 000",
  "3 000",
  "5 000",
  "10 000",
  "15 000",
  "20 000",
  "30 000",
  "50 000",
  "75 000",
  "100 000",
  "150 000",
  "250 000",
  "1 000 000"
]

const quizTopics = [
  "biologi",
  "djur & natur",
  "ekonomi",
  "film & TV",
  "filosofi",
  "fysik",
  "geografi",
  "historia",
  "hälsa & medicin",
  "kemi",
  "konst",
  "litteratur",
  "mat & dryck",
  "matematik",
  "miljö & klimat",
  "mode",
  "musik",
  "myter & legender",
  "psykologi",
  "religion",
  "resor & kultur",
  "rymden & astronomi",
  "samhällskunskap",
  "spel (tv-spel & brädspel)",
  "språk & grammatik",
  "sport",
  "teknik",
  "trafikkunskap"
]

const questions = ref(null)
const index = ref(0)
const selected = ref([])
const attempts = ref(1)
const status = ref(null)
const loading = ref(false)
let subject

// questions.value.forEach((question, index) => question.price = prices[index])

const helpers = ref([
  { name: "50:50", icon: "fa-circle-half-stroke", used: false, callback: fiftyFifty },
  { name: "2 Attempts", icon: "fa-shield", used: false, callback: () => attempts.value++ },
  { name: "Hint", icon: "fa-lightbulb", used: false, callback: hint },
  { name: "Change Question", icon: "fa-rotate", used: false, callback: changeQuestion },
])

const question = computed(() => questions.value[index.value])

const start = async () => {
  loading.value = true

  const subjects = quizTopics.toSorted(() => Math.random() - 0.5).slice(0, 16)
  subject = subjects.pop()

  const input = `
    Generera 15 allmänbildningsfrågor, en för varje ämne i listan: ${subjects.join(", ")}.
    Varje fråga ska ha 4 svarsalternativ där ett svar är korrekt.

    Frågorna ska vara uppdelade i fyra svårighetsgrader:
    - Fråga 1–5: Lätta – frågor om allmänt kända fakta som de flesta vuxna känner till.
    - Fråga 6–10: Medelsvåra – kräver viss skolkunskap eller generell läsförståelse.
    - Fråga 11–14: Svåra – kräver specifik kunskap eller intresse inom området.
    - Fråga 15: Mycket svår – fråga som utmanar även kunniga personer.

    Se till att svårighetsgraden ökar tydligt för varje sektion.
  `

  console.log(subjects)

  const response = await openai.responses.parse({
    model: "gpt-4.1",
    input,
    text: {
      format: zodTextFormat(Quiz, "quiz"),
    },
  })

  questions.value = response.output_parsed.questions.map((question, index) => {
    question.options.sort(() => Math.random() - 0.5)

    return {
      ...question,
      price: prices[index],
      subject: subjects[index]
    }
  })

  loading.value = false
}

const restart = () => {
  questions.value = null
  index.value = 0
  status.value = null
  helpers.value.forEach(helper => helper.used = false)
}

const select = option => {
  selected.value.push(option)

  if (selected.value.length == attempts.value || selected.value.includes(question.value.answer)) {
    setTimeout(() => {

      if (selected.value.includes(question.value.answer)) {
        if (index.value < questions.value.length - 1) {
          index.value++
        }
        else {
          status.value = "win"
          celebrate()
        }
      }
      else {
        status.value = "lose"
      }

      selected.value = []
      attempts.value = 1
    }, 5000)
  }
}

const use = helper => {
  helper.used = true

  if (question.value.helpers) {
    question.value.helpers.push(helper.icon)
  }
  else {
    question.value.helpers = [helper.icon]
  }

  helper.callback()
}

function fiftyFifty() {
  attempts.value += 2

  question.value.options
    .filter(option => option != question.value.answer && !selected.value.includes(option))
    .sort(() => Math.random() - 0.5)
    .slice(0, 2)
    .forEach(option => selected.value.push(option))
}

async function hint() {
  const { output_text } = await openai.responses.create({
    model: "gpt-4.1",
    input: `Ge mig en ledtråd till frågan: ${question.value.text}. Inkludera endast själva ledtråden och se till att den ger tillräckligt med information för att hjälpa spelaren att välja rätt svar. Om du inte kan komma på en bra ledtråd till frågan kan du istället beskriva ett av de felaktiga svarsalternativen.`
  })

  question.value.hint = output_text
}

async function changeQuestion() {
  const Question = z.object({
    text: z.string(),
    options: z.array(z.string()),
    answer: z.string()
  })

  const { output_parsed } = await openai.responses.parse({
    model: "gpt-4.1",
    input: `Ändra frågan '${question.value.text}' och byt ämne till ${subject}, men behåll samma svårighetsgrad.`,
    text: {
      format: zodTextFormat(Question, "question"),
    },
  })

  console.log(newSubject)

  Object.assign(questions.value[index.value], output_parsed, { subject })
}

const celebrate = () => {
  // Confetti
  // const duration = 10 * 1000 // 10 seconds
  // const end = Date.now() + duration
  // const colors = ['#26ccff', '#a25afd', '#ff5e7e', '#88ff5a', '#fcff42', '#ffa62d', '#ff36ff']

  // function randomInRange(min, max) {
  //   return Math.random() * (max - min) + min
  // }

  // (function frame() {
  //   confetti({
  //     particleCount: 1,
  //     origin: { x: Math.random(), y: 0 },
  //     gravity: randomInRange(0.6, 0.8),
  //     ticks: 800,
  //     colors: [colors[Math.floor(Math.random() * colors.length)]]
  //   })

  //   if (Date.now() < end) {
  //     requestAnimationFrame(frame)
  //   }
  // })()


  var end = Date.now() + (15 * 1000)
  var colors = ['#26ccff', '#a25afd', '#ff5e7e', '#88ff5a', '#fcff42', '#ffa62d', '#ff36ff'];

  (function frame() {
    confetti({
      particleCount: 1,
      angle: 60,
      spread: 55,
      decay: 0.95,
      ticks: 400,
      origin: { x: 0 },
      colors: [colors[Math.floor(Math.random() * colors.length)]],
    })
    confetti({
      particleCount: 1,
      angle: 120,
      spread: 55,
      decay: 0.95,
      ticks: 400,
      origin: { x: 1 },
      colors: [colors[Math.floor(Math.random() * colors.length)]]
    })

    if (Date.now() < end) {
      requestAnimationFrame(frame)
    }
  }())


  // Applause
  var audio = new Audio("/sounds/applause.wav")

  audio.play()
}
</script>

<template>
  <div class=" bg-gray-50 h-screen grid place-items-center">
    <div class=" max-w-7xl w-full bg-white shadow-lg text-gray-600">
      <div v-if="questions" class="grid grid-cols-3">
        <div class="col-span-2 p-12 flex flex-col justify-between" :class="{ 'items-center justify-evenly': status }">
          <template v-if="status">
            <header class=" text-center">
              <h2 class=" text-8xl font-bold">{{ status == "win" ? "Congratulations!" : "Game Over" }}</h2>
              <div class=" text-4xl">
                Score:
                <span class=" text-rose-400">
                  {{ status == "win" ? questions.length : index }}/{{ questions.length }}
                </span>
              </div>
            </header>
            <button @click="restart"
              class=" bg-rose-400 text-white px-4 py-2 rounded-md font-semibold cursor-pointer hover:bg-rose-500 transition-colors">
              New Game
            </button>
          </template>
          <template v-else>
            <div class="flex justify-center gap-4">
              <button v-for="helper in helpers" @click.once="use(helper)"
                :disabled="helper.used || selected.length == attempts || selected.includes(question.answer)"
                :class="helper.used ? 'bg-gray-200' : 'bg-rose-400'"
                class=" text-white size-12 rounded-full shadow-md text-xl cursor-pointer disabled:cursor-not-allowed hover:not-disabled:scale-110 hover:not-disabled:shadow-lg transition">
                <i class="fa-solid" :class="helper.icon"></i>
              </button>
            </div>
            <div class=" grid gap-4 justify-items-center text-center">
              <h3 class=" text-gray-400 capitalize">{{ question.subject }}</h3>
              <p class="text-2xl font-semibold text-balance">{{ question.text }}</p>
              <div v-if="question.hint" class=" bg-green-100 text-green-600 p-4 rounded-lg">
                <span class=" font-semibold">Hint:</span> {{ question.hint }}
              </div>
            </div>
            <div class=" grid grid-cols-2 gap-4 select-none">
              <button v-for="option in question.options"
                :disabled="selected.length == attempts || selected.includes(question.answer) || selected.includes(option)"
                @click="select(option)"
                class=" bg-gray-100 px-4 py-2 cursor-pointer disabled:cursor-not-allowed hover:not-disabled:bg-gray-200 transition-colors"
                :class="{ 'bg-green-400 text-white': (selected.length == attempts || selected.includes(question.answer)) && option == question.answer, 'bg-rose-400 text-white': selected.includes(option) && option != question.answer }">
                {{ option }}
              </button>
            </div>
          </template>
        </div>
        <div class="p-12 bg-gray-100 flex flex-col-reverse gap-1">
          <div v-for="({ price, helpers }, i) in questions" :class="{ 'bg-rose-400 text-white': i == index }"
            class=" bg-gray-200 px-4 py-2 grid grid-cols-3">
            <div class="mr-auto" :class="i == index ? 'text-white' : 'text-gray-400'">{{ i + 1 }}</div>
            <div class=" text-center font-semibold">{{ price }}</div>
            <div class=" ml-auto flex items-center gap-2" :class="i == index ? 'text-white' : 'text-rose-400'">
              <i v-for="helper in helpers" class="fa-solid" :class="helper"></i>
            </div>
          </div>
        </div>
      </div>
      <div v-else class=" p-12 flex justify-center">
        <button @click="start" :disabled="loading"
          class=" bg-rose-400 text-white px-4 py-2 cursor-pointer disabled:cursor-not-allowed rounded-md">
          <i v-if="loading" class="fa-solid fa-spinner fa-spin"></i>
          <template v-else>Play</template>
        </button>
      </div>
    </div>
  </div>
</template>