<script setup>
import { ref, computed, onMounted } from 'vue'

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


const SENHAS_VALIDAS = [
  '4821', '7093', '1256', '6634', '9982',
  '3140', '8867', '2519', '5473', '6098',
]

const CHAVE_SORTEIO = 'sorteio-jantar:resultado'
const CHAVE_SENHAS_USADAS = 'sorteio-jantar:senhas-usadas'
const CINCO_DIAS_MS = 5 * 24 * 60 * 60 * 1000

const localAtual = ref('🤔')
const localFinal = ref(null)
const sorteando = ref(false)
const sorteado = ref(false)

const bloqueado = ref(false)
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

onMounted(() => {
  if (SITE_ATIVO) verificarBloqueio()
})

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
  if (sorteando.value) return 'Sorteando...'
  if (bloqueado.value && !desbloqueadoPorSenha.value) return 'Bloqueado'
  return sorteado.value ? 'Sortear novamente' : 'Sortear local'
})

const sortearLocal = () => {
  if (!podeSortear.value) return

  sorteando.value = true
  localFinal.value = null

  const duracao = 1500
  const intervalo = 25

  const timer = setInterval(() => {
    const indice = Math.floor(Math.random() * locais.length)
    localAtual.value = locais[indice]
  }, intervalo)

  setTimeout(() => {
    clearInterval(timer)
    const indiceFinal = Math.floor(Math.random() * locais.length)
    localFinal.value = locais[indiceFinal]
    sorteando.value = false
    sorteado.value = true

    localStorage.setItem(
      CHAVE_SORTEIO,
      JSON.stringify({ local: localFinal.value, timestamp: Date.now() })
    )

    desbloqueadoPorSenha.value = false
    verificarBloqueio()
  }, duracao)
}
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

        <h3 class="text-2xl  text-white mb-5">
          Eles estão curiosos pra saber
        </h3>

        <img src="./assets/images/nozes.jpg" class="rounded-lg my-5" />

        <!-- Display -->
        <div
          class="h-24 flex items-center justify-center
                 rounded-xl bg-black/30 mb-8
                 transition-all duration-300"
          :class="{ 'scale-110 bg-red-500/30': localFinal }"
        >
          <span class="text-3xl font-semibold text-white transition-all duration-200">
            {{ localFinal ?? localAtual }}
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
            Já fomos nesse :( quero sortear novamente
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

