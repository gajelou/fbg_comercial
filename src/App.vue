<script setup lang="ts">
import { onMounted, ref } from "vue";

const API_URL = "https://environment-companies-phi-finance.trycloudflare.com";

type Produto = {
  codigo?: string;
  nome?: string;
  preco?: string;
  imagem?: string;
  link?: string;
  categoriaPrincipal?: string;
  categoria?: string;
};

type GrupoCategoria = {
  categoriaPrincipal: string;
  categorias: string[];
};

const agio = ref<number>(30);
const mostrarPrecos = ref(true);

const vendedorNome = ref("");
const vendedorContato = ref("");
const vendedorInstagram = ref("");

const produtos = ref<Produto[]>([]);
const gruposCategorias = ref<GrupoCategoria[]>([]);
const categoriasPrincipaisSelecionadas = ref<string[]>([]);
const categoriasSelecionadas = ref<string[]>([]);

const carregandoCategorias = ref(false);
const carregando = ref(false);
const mensagem = ref("");
const erro = ref("");
const downloadUrl = ref("");

function normalizarTexto(valor?: string, fallback = ""): string {
  const texto = String(valor ?? "").trim();
  return texto || fallback;
}

function formatarCategoria(valor: string): string {
  return valor
    .replace(/[-_]+/g, " ")
    .replace(/\s+/g, " ")
    .trim();
}

function chaveCategoria(categoriaPrincipal: string, categoria: string): string {
  return `${categoriaPrincipal}|||${categoria}`;
}

function extrairProdutos(dados: unknown): Produto[] {
  if (Array.isArray(dados)) {
    return dados as Produto[];
  }

  if (dados && typeof dados === "object") {
    const objeto = dados as Record<string, unknown>;

    if (Array.isArray(objeto.produtos)) {
      return objeto.produtos as Produto[];
    }

    if (Array.isArray(objeto.data)) {
      return objeto.data as Produto[];
    }
  }

  return [];
}

function montarGruposCategorias(listaProdutos: Produto[]): GrupoCategoria[] {
  const agrupamento = new Map<string, Set<string>>();

  for (const produto of listaProdutos) {
    const categoriaPrincipal = normalizarTexto(
      produto.categoriaPrincipal,
      "SEM-CATEGORIA-PRINCIPAL"
    );

    const categoria = normalizarTexto(
      produto.categoria,
      "SEM-CATEGORIA"
    );

    if (!agrupamento.has(categoriaPrincipal)) {
      agrupamento.set(categoriaPrincipal, new Set<string>());
    }

    agrupamento.get(categoriaPrincipal)?.add(categoria);
  }

  return Array.from(agrupamento.entries())
    .map(([categoriaPrincipal, categorias]) => ({
      categoriaPrincipal,
      categorias: Array.from(categorias).sort((a, b) =>
        a.localeCompare(b, "pt-BR")
      )
    }))
    .sort((a, b) =>
      a.categoriaPrincipal.localeCompare(b.categoriaPrincipal, "pt-BR")
    );
}

function categoriasDoGrupo(grupo: GrupoCategoria): string[] {
  return grupo.categorias.map((categoria) =>
    chaveCategoria(grupo.categoriaPrincipal, categoria)
  );
}

function grupoTotalmenteSelecionado(grupo: GrupoCategoria): boolean {
  const chaves = categoriasDoGrupo(grupo);

  return (
    chaves.length > 0 &&
    chaves.every((chave) => categoriasSelecionadas.value.includes(chave))
  );
}

function grupoParcialmenteSelecionado(grupo: GrupoCategoria): boolean {
  const chaves = categoriasDoGrupo(grupo);
  const quantidadeSelecionada = chaves.filter((chave) =>
    categoriasSelecionadas.value.includes(chave)
  ).length;

  return quantidadeSelecionada > 0 && quantidadeSelecionada < chaves.length;
}

function atualizarCategoriasPrincipaisSelecionadas(): void {
  categoriasPrincipaisSelecionadas.value = gruposCategorias.value
    .filter((grupo) => grupoTotalmenteSelecionado(grupo))
    .map((grupo) => grupo.categoriaPrincipal);
}

function alternarCategoriaPrincipal(
  grupo: GrupoCategoria,
  selecionada: boolean
): void {
  const chavesDoGrupo = categoriasDoGrupo(grupo);

  if (selecionada) {
    categoriasSelecionadas.value = Array.from(
      new Set([...categoriasSelecionadas.value, ...chavesDoGrupo])
    );
  } else {
    categoriasSelecionadas.value = categoriasSelecionadas.value.filter(
      (chave) => !chavesDoGrupo.includes(chave)
    );
  }

  atualizarCategoriasPrincipaisSelecionadas();
}

function atualizarSubcategoria(): void {
  atualizarCategoriasPrincipaisSelecionadas();
}

function selecionarTodasCategorias(): void {
  categoriasSelecionadas.value = gruposCategorias.value.flatMap((grupo) =>
    categoriasDoGrupo(grupo)
  );

  atualizarCategoriasPrincipaisSelecionadas();
}

function limparCategorias(): void {
  categoriasPrincipaisSelecionadas.value = [];
  categoriasSelecionadas.value = [];
}

function obterCategoriasSelecionadasParaEnvio(): string[] {
  return Array.from(
    new Set(
      categoriasSelecionadas.value.map((chave) => {
        const separador = chave.indexOf("|||");
        return separador >= 0 ? chave.slice(separador + 3) : chave;
      })
    )
  );
}

async function lerJsonDaResposta(resposta: Response): Promise<unknown> {
  const texto = await resposta.text();

  if (!texto) {
    return {};
  }

  try {
    return JSON.parse(texto);
  } catch {
    throw new Error("A API retornou uma resposta inválida.");
  }
}

async function carregarAgio(): Promise<void> {
  try {
    const resposta = await fetch(`${API_URL}/agio`);

    if (!resposta.ok) {
      throw new Error("Erro ao carregar ágio atual.");
    }

    const dados = (await lerJsonDaResposta(resposta)) as {
      agioPercentual?: number;
    };

    if (typeof dados.agioPercentual === "number") {
      agio.value = dados.agioPercentual;
    }
  } catch (e) {
    erro.value =
      e instanceof Error ? e.message : "Erro ao carregar ágio atual.";
  }
}

async function carregarProdutos(): Promise<void> {
  carregandoCategorias.value = true;

  try {
    erro.value = "";

    /*
     * Compatibilidade com o backend antigo:
     * 1) tenta carregar produtos completos;
     * 2) se não existir, usa /produtos/categorias;
     * 3) nunca deixa o seletor vazio apenas porque uma rota nova não existe.
     */
    const rotasProdutos = [
      `${API_URL}/produtos`,
      `${API_URL}/produtos.json`
    ];

    let listaProdutos: Produto[] = [];

    for (const rota of rotasProdutos) {
      try {
        const resposta = await fetch(`${rota}?t=${Date.now()}`, {
          cache: "no-store"
        });

        if (!resposta.ok) {
          continue;
        }

        const dados = await lerJsonDaResposta(resposta);
        const produtosEncontrados = extrairProdutos(dados);

        if (produtosEncontrados.length) {
          listaProdutos = produtosEncontrados;
          break;
        }
      } catch {
        // Continua para a próxima rota.
      }
    }

    if (listaProdutos.length) {
      produtos.value = listaProdutos;
      gruposCategorias.value = montarGruposCategorias(listaProdutos);
      selecionarTodasCategorias();
      return;
    }

    // Fallback obrigatório para o backend antigo.
    const respostaCategorias = await fetch(
      `${API_URL}/produtos/categorias?t=${Date.now()}`,
      {
        cache: "no-store"
      }
    );

    if (!respostaCategorias.ok) {
      throw new Error(
        `Erro ao carregar categorias. Status: ${respostaCategorias.status}`
      );
    }

    const dadosCategorias = (await lerJsonDaResposta(
      respostaCategorias
    )) as {
      categorias?: unknown;
    };

    const categoriasAntigas = Array.isArray(dadosCategorias.categorias)
      ? dadosCategorias.categorias
          .filter((categoria): categoria is string =>
            typeof categoria === "string"
          )
          .map((categoria) => categoria.trim())
          .filter(Boolean)
      : [];

    if (!categoriasAntigas.length) {
      throw new Error(
        "O backend não retornou produtos nem categorias."
      );
    }

    /*
     * O endpoint antigo só fornece uma lista plana.
     * Para não quebrar o seletor, exibimos todas em um grupo compatível.
     */
    gruposCategorias.value = [
      {
        categoriaPrincipal: "TODAS-AS-CATEGORIAS",
        categorias: Array.from(new Set(categoriasAntigas)).sort((a, b) =>
          a.localeCompare(b, "pt-BR")
        )
      }
    ];

    produtos.value = [];
    selecionarTodasCategorias();
  } catch (e) {
    produtos.value = [];
    gruposCategorias.value = [];
    limparCategorias();

    erro.value =
      e instanceof Error ? e.message : "Erro ao carregar categorias.";
  } finally {
    carregandoCategorias.value = false;
  }
}

async function gerarCatalogo(): Promise<void> {
  carregando.value = true;
  mensagem.value = "";
  erro.value = "";
  downloadUrl.value = "";

  try {
    if (!gruposCategorias.value.length) {
      throw new Error("Nenhuma categoria foi carregada.");
    }

    if (!categoriasSelecionadas.value.length) {
      throw new Error("Selecione pelo menos uma categoria.");
    }

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
      const dadosErro = await lerJsonDaResposta(atualizarAgio).catch(() => ({}));
      const mensagemErro =
        dadosErro &&
        typeof dadosErro === "object" &&
        "erro" in dadosErro &&
        typeof dadosErro.erro === "string"
          ? dadosErro.erro
          : "Erro ao atualizar ágio.";

      throw new Error(mensagemErro);
    }

    const resposta = await fetch(`${API_URL}/catalogo`, {
      method: "POST",
      headers: {
        "Content-Type": "application/json"
      },
      body: JSON.stringify({
        mostrarPrecos: mostrarPrecos.value,
        vendedorNome: vendedorNome.value.trim(),
        vendedorContato: vendedorContato.value.trim(),
        vendedorInstagram: vendedorInstagram.value.trim(),

        // Novos filtros:
        categoriasPrincipaisSelecionadas:
          categoriasPrincipaisSelecionadas.value,
        categoriasSelecionadas: obterCategoriasSelecionadasParaEnvio()
      })
    });

    const dados = (await lerJsonDaResposta(resposta)) as {
      erro?: string;
      agioPercentual?: number;
      download?: string;
    };

    if (!resposta.ok) {
      throw new Error(dados.erro || "Erro ao gerar catálogo.");
    }

    mensagem.value = mostrarPrecos.value
      ? `Catálogo gerado com ${
          dados.agioPercentual ?? agio.value
        }% de ágio.`
      : "Catálogo sem preços gerado com sucesso.";

    if (dados.download) {
      downloadUrl.value = dados.download.startsWith("http")
        ? dados.download
        : `${API_URL}${dados.download}`;
    }
  } catch (e) {
    erro.value = e instanceof Error ? e.message : "Erro inesperado.";
  } finally {
    carregando.value = false;
  }
}

onMounted(async () => {
  await Promise.all([carregarAgio(), carregarProdutos()]);
});
</script>

<template>
  <main class="page">
    <header class="topbar">
      <div class="brand">
        <div class="logo-mark">FBG</div>

        <div>
          <h1>FBG COMERCIAL</h1>
          <p>GERADOR DE CATÁLOGO ATACADO</p>
        </div>
      </div>
    </header>

    <section class="hero">
      <div class="panel">
        <span class="tag">CONTROLES REMOTOS • FERRAMENTAS</span>

        <h2>Gerar catálogo PDF</h2>

        <p class="description">
          Escolha a porcentagem de ágio, defina se deseja exibir os preços,
          selecione as categorias e informe os dados do vendedor.
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
          <input v-model="mostrarPrecos" type="checkbox" />
          <span>Mostrar preços no catálogo</span>
        </label>

        <div class="form-group">
          <label>Categorias do catálogo</label>

          <div class="category-actions">
            <button
              type="button"
              class="small-button"
              :disabled="carregandoCategorias || !gruposCategorias.length"
              @click="selecionarTodasCategorias"
            >
              Selecionar todas
            </button>

            <button
              type="button"
              class="small-button secondary"
              :disabled="carregandoCategorias || !categoriasSelecionadas.length"
              @click="limparCategorias"
            >
              Limpar
            </button>
          </div>

          <div v-if="carregandoCategorias" class="category-loading">
            Carregando produtos e categorias...
          </div>

          <div
            v-else-if="gruposCategorias.length"
            class="category-groups"
          >
            <section
              v-for="grupo in gruposCategorias"
              :key="grupo.categoriaPrincipal"
              class="category-group"
            >
              <label class="main-category-item">
                <input
                  type="checkbox"
                  :checked="grupoTotalmenteSelecionado(grupo)"
                  :indeterminate="grupoParcialmenteSelecionado(grupo)"
                  @change="
                    alternarCategoriaPrincipal(
                      grupo,
                      ($event.target as HTMLInputElement).checked
                    )
                  "
                />

                <span class="main-category-name">
                  {{ formatarCategoria(grupo.categoriaPrincipal) }}
                </span>

                <small>
                  {{ grupo.categorias.length }}
                  {{ grupo.categorias.length === 1 ? "categoria" : "categorias" }}
                </small>
              </label>

              <div class="subcategory-list">
                <label
                  v-for="categoria in grupo.categorias"
                  :key="chaveCategoria(grupo.categoriaPrincipal, categoria)"
                  class="subcategory-item"
                >
                  <input
                    v-model="categoriasSelecionadas"
                    type="checkbox"
                    :value="chaveCategoria(grupo.categoriaPrincipal, categoria)"
                    @change="atualizarSubcategoria"
                  />

                  <span>{{ formatarCategoria(categoria) }}</span>
                </label>
              </div>
            </section>
          </div>

          <p v-else class="category-empty">
            Nenhuma categoria encontrada. Execute o scraper primeiro.
          </p>
        </div>

        <div class="form-group">
          <label>Nome do vendedor</label>
          <input
            v-model="vendedorNome"
            class="text-input"
            type="text"
            placeholder="José da Silva"
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
            placeholder="Ex: ze@fbg.com"
          />
        </div>

        <button
          class="primary"
          :disabled="
            carregando ||
            carregandoCategorias ||
            !categoriasSelecionadas.length
          "
          @click="gerarCatalogo"
        >
          <span v-if="carregando" class="loading-content">
            <span class="tool-loader">🔧</span>
            GERANDO CATÁLOGO...
          </span>

          <span v-else>GERAR CATÁLOGO</span>
        </button>

        <div v-if="carregando" class="loading-box">
          <div class="tools-animation">
            <span>▶</span>
            <span>🔧</span>
            <span>🪛</span>
          </div>

          <p>Montando seu catálogo de produtos...</p>
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
          rel="noopener noreferrer"
        >
          BAIXAR CATÁLOGO PDF
        </a>
      </div>
    </section>
  </main>
</template>

<style scoped>
* {
  box-sizing: border-box;
}

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
  max-width: 560px;
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

.form-group > label {
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
  background: #fff;
}

.input-wrap input {
  flex: 1;
  min-width: 0;
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

.text-input:focus,
.input-wrap:focus-within {
  border-color: #d4aa00;
  box-shadow: 0 0 0 3px rgba(246, 196, 0, 0.18);
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

.category-actions {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  margin-bottom: 12px;
}

.small-button {
  border: 0;
  border-radius: 6px;
  background: #111;
  color: #f6c400;
  padding: 9px 12px;
  font-size: 11px;
  font-weight: 900;
  text-transform: uppercase;
  cursor: pointer;
}

.small-button.secondary {
  background: #e7e7e7;
  color: #222;
}

.small-button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.category-groups {
  display: flex;
  flex-direction: column;
  gap: 14px;
  max-height: 500px;
  overflow-y: auto;
  padding: 2px 5px 2px 2px;
}

.category-group {
  border: 1px solid #d7d7d7;
  border-radius: 9px;
  overflow: hidden;
  background: #fff;
}

.main-category-item {
  display: flex;
  align-items: center;
  gap: 9px;
  margin: 0;
  padding: 12px;
  background: #111;
  color: #f6c400;
  cursor: pointer;
}

.main-category-item input {
  width: 18px;
  height: 18px;
  flex-shrink: 0;
  cursor: pointer;
}

.main-category-name {
  flex: 1;
  min-width: 0;
  font-size: 13px;
  line-height: 1.3;
  font-weight: 900;
  text-transform: uppercase;
  overflow-wrap: anywhere;
}

.main-category-item small {
  color: #fff;
  font-size: 10px;
  font-weight: 700;
  white-space: nowrap;
}

.subcategory-list {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 9px 14px;
  padding: 13px;
  background: #fafafa;
}

.subcategory-item {
  display: flex;
  align-items: flex-start;
  gap: 7px;
  min-width: 0;
  margin: 0;
  cursor: pointer;
}

.subcategory-item input {
  width: 16px;
  height: 16px;
  margin-top: 1px;
  flex-shrink: 0;
  cursor: pointer;
}

.subcategory-item span {
  font-size: 10px;
  line-height: 1.35;
  font-weight: 700;
  text-transform: uppercase;
  overflow-wrap: anywhere;
  word-break: break-word;
}

.category-loading,
.category-empty {
  padding: 13px;
  border-radius: 8px;
  font-size: 13px;
  text-align: center;
}

.category-loading {
  background: #f7f7f7;
  border: 1px dashed #aaa;
  color: #444;
}

.category-empty {
  background: #fff8d6;
  border: 1px solid #f6c400;
  color: #6b5700;
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
  0%,
  100% {
    transform: translateY(0) rotate(0deg);
  }

  50% {
    transform: translateY(-8px) rotate(-12deg);
  }
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
    font-size: 20px;
    padding: 13px;
  }

  .input-wrap span {
    padding: 14px 16px;
    font-size: 18px;
  }

  .subcategory-list {
    grid-template-columns: 1fr;
  }

  .main-category-item {
    align-items: flex-start;
  }

  .main-category-item small {
    white-space: normal;
    text-align: right;
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

  .form-group > label {
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

  .main-category-item {
    padding: 10px;
  }

  .main-category-name {
    font-size: 11px;
  }

  .main-category-item small {
    font-size: 9px;
  }

  .subcategory-list {
    padding: 11px;
  }

  .subcategory-item span {
    font-size: 9px;
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
