<script setup>
import { ref, computed, onMounted, nextTick } from 'vue'

const SITE_ATIVO = true

const locais = [
  'Harry Potter',
  'Hotto Bento',
  'Restaurante do pirata',
  'Ramen',
  'Mercearia São José',
  'Pizza Hit',
  'Portugal em Casa',
  'Bar Jazz',
  'Jotaka',
  'Street Burguer',
  'Raspadinha Japonesa',
  'Nakombinha',
  'Café Encantado',
  'Mr Jack',
  'Padaria Marcos',
  'Salz Burguer',
  'Colha',
]

const CORES = locais.map(
  (_, i) => `hsl(${Math.round((360 / locais.length) * i)}, 65%, 42%)`
)

const SENHAS_VALIDAS = [
  '4821', '7093', '1256', '6634', '9982',
  '3140', '8867', '2519', '5473', '6098',
]

const CHAVE_SORTEIO = 'sorteio-jantar:resultado'
const CHAVE_SENHAS_USADAS = 'sorteio-jantar:senhas-usadas'
const CINCO_DIAS_MS = 5 * 24 * 60 * 60 * 1000

const canvasRef = ref(null)
const CANVAS_SIZE = 288
const accumulatedRotation = ref(0)

const IDLE_BLUR = 4
const REVEAL_DURATION_MS = 4000 // quanto tempo fica legível após o giro
const blurPx = ref(IDLE_BLUR)


const localFinal = ref(null)
const sorteando = ref(false)
const sorteado = ref(false)

const bloqueado = ref(false)
const bypass = ref(false)
const diasRestantes = ref(0)
const desbloqueadoPorSenha = ref(false)
const senhaInput = ref('')
const erroSenha = ref('')

function getSenhasUsadas() {
  return JSON.parse(localStorage.getItem(CHAVE_SENHAS_USADAS) || '[]')
}

function marcarSenhaUsada(senha) {
  const usadas = getSenhasUsadas()
  usadas.push(senha)
  localStorage.setItem(CHAVE_SENHAS_USADAS, JSON.stringify(usadas))
}

function verificarBloqueio() {
  if (bypass.value) { bloqueado.value = false; return }
  const dados = JSON.parse(localStorage.getItem(CHAVE_SORTEIO) || 'null')
  if (dados) {
    const passou = Date.now() - dados.timestamp
    if (passou < CINCO_DIAS_MS) {
      bloqueado.value = true
      diasRestantes.value = Math.ceil((CINCO_DIAS_MS - passou) / (24 * 60 * 60 * 1000))
      localFinal.value = dados.local
      sorteado.value = true
      return
    }
  }
  bloqueado.value = false
}

function liberarComSenha() {
  const senha = senhaInput.value.trim()
  erroSenha.value = ''

  if (!/^\d{4}$/.test(senha) || !SENHAS_VALIDAS.includes(senha)) {
    erroSenha.value = 'Senha inválida'
    return
  }

  const usadas = getSenhasUsadas()
  if (usadas.includes(senha)) {
    erroSenha.value = 'Essa senha já foi usada'
    return
  }

  marcarSenhaUsada(senha)
  desbloqueadoPorSenha.value = true
  senhaInput.value = ''
}

const podeSortear = computed(
  () => !sorteando.value && (!bloqueado.value || desbloqueadoPorSenha.value)
)

const rotuloBotao = computed(() => {
  if (sorteando.value) return 'Girando...'
  if (bloqueado.value && !desbloqueadoPorSenha.value) return 'Bloqueado'
  return sorteado.value ? 'Girar novamente' : 'Girar a roleta'
})

// ---------- Desenho da roleta ----------
function drawWheel(rotation) {
  const canvas = canvasRef.value
  if (!canvas) return
  const ctx = canvas.getContext('2d')
  const size = canvas.width
  const cx = size / 2
  const cy = size / 2
  const radius = size / 2 - 6
  const segAngle = (2 * Math.PI) / locais.length

  ctx.clearRect(0, 0, size, size)
  ctx.save()
  ctx.translate(cx, cy)
  ctx.rotate(rotation)

  locais.forEach((nome, i) => {
    const start = i * segAngle
    const end = start + segAngle

    ctx.beginPath()
    ctx.moveTo(0, 0)
    ctx.arc(0, 0, radius, start, end)
    ctx.closePath()
    ctx.fillStyle = CORES[i]
    ctx.fill()
    ctx.strokeStyle = 'rgba(255,255,255,0.15)'
    ctx.lineWidth = 1
    ctx.stroke()

    ctx.save()
    ctx.rotate(start + segAngle / 2)
    ctx.textAlign = 'right'
    ctx.textBaseline = 'middle'
    ctx.fillStyle = '#fff'
    ctx.font = '600 11px system-ui, sans-serif'
    const maxWidth = radius - 26
    let texto = nome
    while (ctx.measureText(texto).width > maxWidth && texto.length > 3) {
      texto = texto.slice(0, -1)
    }
    if (texto !== nome) texto = texto.slice(0, -1) + '…'
    ctx.fillText(texto, radius - 10, 0)
    ctx.restore()
  })

  ctx.restore()

  ctx.beginPath()
  ctx.arc(cx, cy, radius, 0, 2 * Math.PI)
  ctx.strokeStyle = 'rgba(255,255,255,0.4)'
  ctx.lineWidth = 4
  ctx.stroke()
}

// ---------- Física do giro ----------
function animarGiro(rotInicial, rotFinal, duracao, finalIndex) {
  const inicioTempo = performance.now()

  function frame(agora) {
    const decorrido = agora - inicioTempo
    const t = Math.min(decorrido / duracao, 1)

    const eased = 1 - Math.pow(1 - t, 5)
    const rotacaoAtual = rotInicial + (rotFinal - rotInicial) * eased
    accumulatedRotation.value = rotacaoAtual
    drawWheel(rotacaoAtual)

    if (t < 1) {
      // blur nunca cai abaixo do IDLE_BLUR enquanto ainda está girando —
      // só some de verdade no frame final, junto da revelação
      const velocidade = 5 * Math.pow(1 - t, 4)
      blurPx.value = Math.max(velocidade * 1.8, IDLE_BLUR)
      requestAnimationFrame(frame)
    } else {
      blurPx.value = 0 // revela o resultado por alguns segundos
      sorteando.value = false
      sorteado.value = true
      localFinal.value = locais[finalIndex]

      localStorage.setItem(
        CHAVE_SORTEIO,
        JSON.stringify({ local: localFinal.value, timestamp: Date.now() })
      )

      desbloqueadoPorSenha.value = false
      verificarBloqueio()

      // depois de um tempo, borra de novo (o resultado continua no card de texto)
      setTimeout(() => {
        blurPx.value = IDLE_BLUR
      }, REVEAL_DURATION_MS)
    }
  }

  requestAnimationFrame(frame)
}
const sortearLocal = () => {
  if (!podeSortear.value) return

  sorteando.value = true
  localFinal.value = null

  const segAngle = (2 * Math.PI) / locais.length
  const finalIndex = Math.floor(Math.random() * locais.length)

  const anguloPonteiro = -Math.PI / 2
  let offset = anguloPonteiro - (finalIndex + 0.5) * segAngle
  offset = ((offset % (2 * Math.PI)) + 2 * Math.PI) % (2 * Math.PI)

  const jitter = (Math.random() - 0.5) * segAngle * 0.6

  const voltas = 6 + Math.floor(Math.random() * 3)
  const rotacaoTotal = voltas * 2 * Math.PI + offset + jitter

  const rotInicial = accumulatedRotation.value
  const rotFinal = rotInicial + rotacaoTotal
  const duracao = 4200 + Math.random() * 800

  animarGiro(rotInicial, rotFinal, duracao, finalIndex)
}

onMounted(async () => {
  if (!SITE_ATIVO) return
  verificarBloqueio()
  await nextTick()
  drawWheel(accumulatedRotation.value)
})
</script>

<template>
  <div class="min-h-screen bg-gradient-to-br from-red-900 to-slate-700
              flex items-center justify-center px-4">

    <div class="w-full max-w-md bg-white/10 backdrop-blur-lg
               rounded-2xl p-8 text-center shadow-2xl">

      <!-- Site inativo: nenhum encontro planejado -->
      <template v-if="!SITE_ATIVO">
        <h1 class="text-3xl font-bold text-white mb-4">
          Onde vamos hoje?
        </h1>
        <img src="./assets/images/nozes.jpg" class="rounded-lg my-5" />
        <p class="text-white/80 text-lg mt-4">
          Nenhum date planejado.
          <br>
          Estamos aguardando o próximo
        </p>
      </template>

      <!-- Site ativo: fluxo normal -->
      <template v-else>
        <h1 class="text-3xl font-bold text-white mb-8">
          Onde vamos hoje?
        </h1>

        <h3 class="text-xl  text-white mb-5">
          Eles estão curiosos pra saber
        </h3>

        <img src="./assets/images/nozes.jpg" class="rounded-lg my-5" />

        <!-- Roleta -->
        <div class="relative w-72 h-72 mx-auto mb-6">
          <div
            class="absolute left-1/2 -translate-x-1/2 -top-3 z-10 w-0 h-0
                   border-l-[9px] border-l-transparent
                   border-r-[9px] border-r-transparent
                   border-t-[18px] border-t-yellow-400"
          />
          <canvas
            ref="canvasRef"
            :width="CANVAS_SIZE"
            :height="CANVAS_SIZE"
            class="rounded-full shadow-2xl bg-black/30"
            :style="{
              filter: `blur(${blurPx}px)`,
              transition: sorteando ? 'none' : 'filter 0.2s ease-out'
            }"
          />
        </div>

        <!-- Resultado -->
        <div
          class="h-24 flex items-center justify-center
                 rounded-xl bg-black/30 mb-8
                 transition-all duration-300"
          :class="{ 'scale-110 bg-red-500/30': localFinal && !sorteando }"
        >
          <span class="text-3xl font-semibold text-white transition-all duration-200">
            {{ sorteando ? 'Girando...' : (localFinal ?? '🤔') }}
          </span>
        </div>

        <!-- Campo de senha (só aparece se bloqueado) -->
        <div v-if="bloqueado && !desbloqueadoPorSenha" class="mb-4">
          <input
            v-model="senhaInput"
            type="text"
            maxlength="4"
            inputmode="numeric"
            placeholder="Senha de 4 dígitos"
            class="w-full py-2 px-4 rounded-xl bg-white/20 text-white
                   placeholder-white/50 text-center mb-2 outline-none"
            @keyup.enter="liberarComSenha"
          />
          <button
            @click="liberarComSenha"
            class="w-full py-2 rounded-xl font-semibold
                   border border-white text-white
                   hover:bg-white/10 transition-all"
          >
            Já fomos nesse :( quero sortear de novo
          </button>
          <p v-if="erroSenha" class="text-red-300 text-sm mt-2">{{ erroSenha }}</p>
        </div>

        <!-- Botão principal -->
        <button
          @click="sortearLocal"
          :disabled="!podeSortear"
          class="w-full py-3 rounded-xl font-bold text-lg
                 transition-all duration-300
                 bg-emerald-500 hover:bg-emerald-400 text-white
                 disabled:opacity-50 disabled:cursor-not-allowed"
        >
          {{ rotuloBotao }}
        </button>
      </template>

    </div>
  </div>
</template>