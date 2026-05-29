<script setup lang="ts">
import { ref, onMounted } from "vue";

const API_URL = "https://county-common-latitude-system.trycloudflare.com";

const agio = ref<number>(30);
const mostrarPrecos = ref(true);

const vendedorNome = ref("");
const vendedorContato = ref("");
const vendedorInstagram = ref("");

const carregando = ref(false);
const mensagem = ref("");
const erro = ref("");
const downloadUrl = ref("");

async function carregarAgio() {
  try {
    const resposta = await fetch(`${API_URL}/agio`);
    const dados = await resposta.json();

    agio.value = dados.agioPercentual;
  } catch {
    erro.value = "Erro ao carregar ágio atual.";
  }
}

async function gerarCatalogo() {
  carregando.value = true;
  mensagem.value = "";
  erro.value = "";
  downloadUrl.value = "";

  try {
    const atualizarAgio = await fetch(`${API_URL}/agio`, {
      method: "POST",
      headers: {
        "Content-Type": "application/json"
      },
      body: JSON.stringify({
        agioPercentual: agio.value
      })
    });

    if (!atualizarAgio.ok) {
      throw new Error("Erro ao atualizar ágio.");
    }

    const resposta = await fetch(`${API_URL}/catalogo`, {
      method: "POST",
      headers: {
        "Content-Type": "application/json"
      },
      body: JSON.stringify({
        mostrarPrecos: mostrarPrecos.value,
        vendedorNome: vendedorNome.value,
        vendedorContato: vendedorContato.value,
        vendedorInstagram: vendedorInstagram.value
      })
    });

    const dados = await resposta.json();

    if (!resposta.ok) {
      throw new Error(dados.erro || "Erro ao gerar catálogo.");
    }

    mensagem.value = mostrarPrecos.value
      ? `Catálogo gerado com ${dados.agioPercentual}% de ágio.`
      : "Catálogo sem preços gerado com sucesso.";

    downloadUrl.value = `${API_URL}${dados.download}`;
  } catch (e) {
    erro.value = e instanceof Error ? e.message : "Erro inesperado.";
  } finally {
    carregando.value = false;
  }
}

onMounted(carregarAgio);
</script>

<template>
  <main class="page">
    <header class="topbar">
      <div class="brand">
        <div class="logo-mark">C&B</div>

        <div>
          <h1>CAMARGO & BARROS</h1>
          <p>GERADOR DE CATÁLOGO ATACADO</p>
        </div>
      </div>
    </header>

    <section class="hero">
      <div class="panel">
        <span class="tag">FERRAMENTAS • VARIEDADES • ATACADO</span>

        <h2>Gerar catálogo PDF</h2>

        <p class="description">
          Escolha a porcentagem de ágio, defina se deseja exibir os preços e informe os dados do vendedor.
        </p>

        <div class="form-group">
          <label>Ágio sobre o valor de atacado</label>

          <div class="input-wrap">
            <input
              v-model.number="agio"
              type="number"
              min="0"
              step="1"
              placeholder="30"
            />

            <span>%</span>
          </div>
        </div>

        <label class="checkbox">
          <input
            v-model="mostrarPrecos"
            type="checkbox"
          />

          <span>Mostrar preços no catálogo</span>
        </label>

        <div class="form-group">
          <label>Nome do vendedor</label>
          <input
            v-model="vendedorNome"
            class="text-input"
            type="text"
            placeholder="Camarguinho"
          />
        </div>

        <div class="form-group">
          <label>Contato / WhatsApp</label>
          <input
            v-model="vendedorContato"
            class="text-input"
            type="text"
            placeholder="Ex: (11) 99999-9999"
          />
        </div>

        <div class="form-group">
          <label>Email opcional</label>
          <input
            v-model="vendedorInstagram"
            class="text-input"
            type="text"
            placeholder="Ex: camarguinho@camargoebarros.com"
          />
        </div>

        <button
          class="primary"
          @click="gerarCatalogo"
          :disabled="carregando"
        >
          <span v-if="carregando" class="loading-content">
            <span class="tool-loader">🔧</span>
            GERANDO CATÁLOGO...
          </span>

          <span v-else>
            GERAR CATÁLOGO
          </span>
        </button>

        <div v-if="carregando" class="loading-box">
          <div class="tools-animation">
            <span>🔨</span>
            <span>🔧</span>
            <span>🪛</span>
          </div>

          <p>Montando seu catálogo de ferramentas...</p>
          <small>Isso pode levar alguns instantes.</small>
        </div>

        <p v-if="mensagem" class="success">
          {{ mensagem }}
        </p>

        <p v-if="erro" class="error">
          {{ erro }}
        </p>

        <a
          v-if="downloadUrl"
          :href="downloadUrl"
          class="download"
          target="_blank"
        >
          BAIXAR CATÁLOGO PDF
        </a>
      </div>
    </section>
  </main>
</template>

<style scoped>
.page {
  min-height: 100vh;
  background: #f5f5f5;
  font-family: Arial, Helvetica, sans-serif;
  color: #222;
}

.topbar {
  background: #111;
  color: #fff;
  padding: 18px 24px;
  border-bottom: 5px solid #f6c400;
}

.brand {
  max-width: 980px;
  margin: 0 auto;
  display: flex;
  align-items: center;
  gap: 14px;
}

.logo-mark {
  width: 58px;
  height: 58px;
  border-radius: 10px;
  background: #f6c400;
  color: #111;
  display: grid;
  place-items: center;
  font-weight: 900;
  font-size: 19px;
}

.brand h1 {
  margin: 0;
  font-size: 24px;
  letter-spacing: 1px;
}

.brand p {
  margin: 3px 0 0;
  font-size: 12px;
  color: #f6c400;
  font-weight: bold;
  letter-spacing: 1px;
}

.hero {
  max-width: 980px;
  margin: 44px auto;
  padding: 0 20px;
}

.panel {
  max-width: 480px;
  margin: 0 auto;
  background: #fff;
  border-radius: 10px;
  padding: 30px;
  border: 1px solid #ddd;
  box-shadow: 0 8px 22px rgba(0, 0, 0, 0.08);
}

.tag {
  display: inline-block;
  background: #f6c400;
  color: #111;
  font-size: 11px;
  font-weight: 900;
  padding: 7px 10px;
  border-radius: 4px;
  margin-bottom: 16px;
  letter-spacing: 0.5px;
}

h2 {
  margin: 0;
  font-size: 30px;
  text-transform: uppercase;
  color: #111;
}

.description {
  color: #555;
  font-size: 15px;
  line-height: 1.5;
  margin: 12px 0 24px;
}

.form-group {
  margin-bottom: 18px;
}

.form-group label {
  display: block;
  font-weight: 800;
  text-transform: uppercase;
  font-size: 13px;
  margin-bottom: 8px;
}

.input-wrap {
  display: flex;
  align-items: center;
  border: 2px solid #111;
  border-radius: 8px;
  overflow: hidden;
  background: white;
}

.input-wrap input {
  flex: 1;
  border: 0;
  padding: 15px;
  font-size: 22px;
  font-weight: bold;
  outline: none;
}

.input-wrap span {
  background: #111;
  color: #f6c400;
  padding: 16px 18px;
  font-size: 20px;
  font-weight: 900;
}

.text-input {
  width: 100%;
  border: 2px solid #111;
  border-radius: 8px;
  padding: 14px;
  font-size: 16px;
  font-weight: bold;
  outline: none;
}

.checkbox {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 22px;
  font-weight: 800;
  color: #222;
  cursor: pointer;
}

.checkbox input {
  width: 18px;
  height: 18px;
  cursor: pointer;
}

.primary {
  width: 100%;
  padding: 15px;
  background: #111;
  color: #f6c400;
  border: 0;
  border-radius: 8px;
  font-weight: 900;
  font-size: 15px;
  cursor: pointer;
  letter-spacing: 0.7px;
}

.primary:hover {
  background: #222;
}

.primary:disabled {
  background: #999;
  color: #eee;
  cursor: not-allowed;
}

.download {
  display: block;
  margin-top: 16px;
  text-align: center;
  background: #f6c400;
  color: #111;
  padding: 15px;
  border-radius: 8px;
  text-decoration: none;
  font-weight: 900;
}

.success {
  margin-top: 16px;
  padding: 12px;
  background: #ecfdf3;
  color: #166534;
  border: 1px solid #bbf7d0;
  border-radius: 8px;
  text-align: center;
  font-weight: bold;
}

.error {
  margin-top: 16px;
  padding: 12px;
  background: #fef2f2;
  color: #991b1b;
  border: 1px solid #fecaca;
  border-radius: 8px;
  text-align: center;
  font-weight: bold;
}

.loading-content {
  display: inline-flex;
  align-items: center;
  gap: 8px;
}

.tool-loader {
  display: inline-block;
  animation: spinTool 1s linear infinite;
}

.loading-box {
  margin-top: 18px;
  padding: 18px;
  background: #fff8d6;
  border: 2px dashed #f6c400;
  border-radius: 10px;
  text-align: center;
}

.tools-animation {
  display: flex;
  justify-content: center;
  gap: 14px;
  font-size: 30px;
  margin-bottom: 10px;
}

.tools-animation span {
  display: inline-block;
  animation: bounceTool 1s ease-in-out infinite;
}

.tools-animation span:nth-child(2) {
  animation-delay: 0.15s;
}

.tools-animation span:nth-child(3) {
  animation-delay: 0.3s;
}

.loading-box p {
  margin: 0;
  font-weight: 900;
  color: #111;
}

.loading-box small {
  display: block;
  margin-top: 5px;
  color: #555;
}

@keyframes spinTool {
  from {
    transform: rotate(0deg);
  }

  to {
    transform: rotate(360deg);
  }
}

@keyframes bounceTool {
  0%, 100% {
    transform: translateY(0) rotate(0deg);
  }

  50% {
    transform: translateY(-8px) rotate(-12deg);
  }
}

* {
  box-sizing: border-box;
}

@media (max-width: 768px) {
  .topbar {
    padding: 14px 16px;
  }

  .brand {
    max-width: 100%;
    gap: 10px;
  }

  .logo-mark {
    width: 48px;
    height: 48px;
    font-size: 16px;
    flex-shrink: 0;
  }

  .brand h1 {
    font-size: 18px;
    line-height: 1.1;
  }

  .brand p {
    font-size: 10px;
    letter-spacing: 0.5px;
  }

  .hero {
    margin: 24px auto;
    padding: 0 14px;
  }

  .panel {
    max-width: 100%;
    padding: 22px 18px;
    border-radius: 8px;
  }

  .tag {
    font-size: 10px;
    line-height: 1.3;
  }

  h2 {
    font-size: 24px;
    line-height: 1.1;
  }

  .description {
    font-size: 14px;
  }

  .input-wrap input {
    min-width: 0;
    font-size: 20px;
    padding: 13px;
  }

  .input-wrap span {
    padding: 14px 16px;
    font-size: 18px;
  }

  .primary,
  .download {
    font-size: 14px;
    padding: 14px 12px;
  }

  .checkbox {
    align-items: flex-start;
    font-size: 14px;
    line-height: 1.3;
  }

  .loading-box {
    padding: 14px;
  }

  .tools-animation {
    font-size: 26px;
  }
}

@media (max-width: 420px) {
  .brand {
    align-items: flex-start;
  }

  .logo-mark {
    width: 42px;
    height: 42px;
    font-size: 14px;
  }

  .brand h1 {
    font-size: 16px;
  }

  .brand p {
    font-size: 9px;
  }

  .hero {
    margin: 18px auto;
    padding: 0 10px;
  }

  .panel {
    padding: 18px 14px;
  }

  h2 {
    font-size: 21px;
  }

  .tag {
    font-size: 9px;
    padding: 6px 8px;
  }

  .description {
    font-size: 13px;
  }

  .form-group label {
    font-size: 12px;
  }

  .input-wrap input {
    font-size: 18px;
    padding: 12px;
  }

  .input-wrap span {
    font-size: 17px;
    padding: 13px 14px;
  }

  .primary,
  .download {
    font-size: 13px;
    letter-spacing: 0.3px;
  }

  .success,
  .error {
    font-size: 13px;
    padding: 10px;
  }
}
</style>