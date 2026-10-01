<template>
  <h1>Cadastro de Carros</h1>
  <h3>
    {{ editandoId ? "Editando veículo" : "Adicione um novo carro à lista:" }}
  </h3>

  <label for="marca" style="font-weight: bold">Marca:</label><br />
  <input
    type="text"
    id="marca"
    v-model="form.marca"
    placeholder="Marca do carro"
  /><br />
  <label for="modelo" style="font-weight: bold">Modelo:</label><br />
  <input
    type="text"
    id="modelo"
    v-model="form.modelo"
    placeholder="Modelo do carro"
  /><br />
  <label for="ano" style="font-weight: bold">Ano:</label><br />
  <input
    type="number"
    id="ano"
    v-model.number="form.ano"
    :min="ANO_MINIMO"
    :max="anoMaximo"
    step="1"
    oninput="this.value = this.value.slice(0, 4)"
    placeholder="Ano do carro"
  /><br />
  <label for="preco" style="font-weight: bold">Preço:</label><br />
  <input
    type="text"
    id="preco"
    inputmode="numeric"
    :value="precoTexto"
    @input="digitarPreco"
    placeholder="R$ 0,00"
  /><br />

  <button @click="salvar" :disabled="carregando">
    {{ editandoId ? "Salvar Alterações" : "Salvar" }}
  </button>
  <button v-if="editandoId" @click="limparFormulario">Cancelar</button>

  <p v-if="carregando">Carregando...</p>

  <div
    v-if="mensagem"
    class="toast"
    :class="mensagem.tipo"
    @click="mensagem = null"
  >
    {{ mensagem.texto }}
  </div>

  <table class="lista">
    <thead>
      <tr>
        <th>Marca</th>
        <th>Modelo</th>
        <th>Ano</th>
        <th>Preço</th>
        <th>Ações</th>
      </tr>
    </thead>
    <tbody>
      <Card
        v-for="veiculo in carros"
        :key="veiculo.id"
        :carro="veiculo"
        @editar="editar"
        @excluir="excluir"
      />
      <tr v-if="!carregando && carros.length === 0">
        <td colspan="5">Nenhum veículo cadastrado.</td>
      </tr>
    </tbody>
  </table>
</template>

<script setup>
import Card from "./componentes/card.vue";
import axios from "axios";
import { ref, onMounted } from "vue";

const api = axios.create({ baseURL: "https://api-lpv.onrender.com" });

const carros = ref([]);
const form = ref({ marca: "", modelo: "", ano: null, preco: null });
const editandoId = ref(null);
const carregando = ref(false);
const mensagem = ref(null);
let timerMensagem = null;
const precoTexto = ref("");

const nomesCampos = {
  marca: "Marca",
  modelo: "Modelo",
  ano: "Ano",
  preco: "Preço",
};

const ANO_MINIMO = 1886;
const anoMaximo = new Date().getFullYear() + 1;

function juntar(lista) {
  if (lista.length === 1) return lista[0];
  return lista.slice(0, -1).join(", ") + " e " + lista[lista.length - 1];
}

function formatarMoeda(valor) {
  return valor.toLocaleString("pt-BR", { style: "currency", currency: "BRL" });
}

function digitarPreco(evento) {
  const digitos = evento.target.value.replace(/\D/g, "");
  if (!digitos) {
    form.value.preco = null;
    precoTexto.value = "";
  } else {
    form.value.preco = Number(digitos) / 100;
    precoTexto.value = formatarMoeda(form.value.preco);
  }
  evento.target.value = precoTexto.value;
}

function mostrarMensagem(texto, tipo = "erro") {
  mensagem.value = { texto, tipo };
  clearTimeout(timerMensagem);
  timerMensagem = setTimeout(() => (mensagem.value = null), 4000);
}

async function listar() {
  carregando.value = true;
  try {
    const resposta = await api.get("/veiculos");
    carros.value = resposta.data.data;
  } catch (e) {
    mostrarMensagem(
      "Erro ao carregar veículos: " + (e.response?.data?.error || e.message),
    );
  } finally {
    carregando.value = false;
  }
}

async function salvar() {
  const dados = {
    marca: form.value.marca.trim(),
    modelo: form.value.modelo.trim(),
    ano: form.value.ano,
    preco: form.value.preco,
  };

  const vazios = Object.keys(nomesCampos)
    .filter((campo) => !dados[campo])
    .map((campo) => nomesCampos[campo]);
  if (vazios.length === 1) {
    mostrarMensagem("Preencha o campo: " + vazios[0] + ".");
    return;
  }
  if (vazios.length > 1) {
    mostrarMensagem("Preencha os campos: " + juntar(vazios) + ".");
    return;
  }

  const soNumeros = ["marca", "modelo"]
    .filter((campo) => /^\d+$/.test(dados[campo]))
    .map((campo) => nomesCampos[campo]);
  if (soNumeros.length === 1) {
    mostrarMensagem(soNumeros[0] + " não pode ter apenas números.");
    return;
  }
  if (soNumeros.length > 1) {
    mostrarMensagem(juntar(soNumeros) + " não podem ter apenas números.");
    return;
  }

  if (
    !Number.isInteger(dados.ano) ||
    dados.ano < ANO_MINIMO ||
    dados.ano > anoMaximo
  ) {
    mostrarMensagem(
      "Ano inválido: informe um ano entre " +
        ANO_MINIMO +
        " e " +
        anoMaximo +
        ".",
    );
    return;
  }

  carregando.value = true;
  try {
    if (editandoId.value) {
      await api.patch(`/veiculos/${editandoId.value}`, dados);
      mostrarMensagem("Veículo atualizado!", "sucesso");
    } else {
      await api.post("/veiculos", dados);
      mostrarMensagem("Veículo cadastrado!", "sucesso");
    }
    limparFormulario();
    await listar();
  } catch (e) {
    mostrarMensagem(
      "Erro ao salvar: " + (e.response?.data?.error || e.message),
    );
  } finally {
    carregando.value = false;
  }
}

async function editar(id) {
  try {
    const resposta = await api.get(`/veiculos/${id}`);
    const { marca, modelo, ano, preco } = resposta.data;
    form.value = { marca, modelo, ano, preco };
    precoTexto.value = formatarMoeda(preco);
    editandoId.value = id;
  } catch (e) {
    mostrarMensagem(
      "Erro ao buscar veículo: " + (e.response?.data?.error || e.message),
    );
  }
}

// DELETE /veiculos/{id}
async function excluir(id) {
  if (!confirm("Deseja remover este veículo?")) return;
  try {
    await api.delete(`/veiculos/${id}`);
    if (editandoId.value === id) limparFormulario();
    mostrarMensagem("Veículo removido!", "sucesso");
    await listar();
  } catch (e) {
    mostrarMensagem(
      "Erro ao excluir: " + (e.response?.data?.error || e.message),
    );
  }
}

function limparFormulario() {
  form.value = { marca: "", modelo: "", ano: null, preco: null };
  precoTexto.value = "";
  editandoId.value = null;
}

onMounted(listar);
</script>

<style scoped>
.lista {
  width: 100%;
  border-collapse: collapse;
  margin-top: 1rem;
}

.lista th {
  background: #333;
  color: white;
  text-align: left;
  padding: 8px;
  border: 1px solid black;
}

.lista td {
  padding: 8px;
  border: 1px solid black;
}

button {
  margin-top: 0.5rem;
  margin-right: 0.5rem;
}

.toast {
  position: fixed;
  top: 20px;
  right: 20px;
  padding: 12px 20px;
  border-radius: 6px;
  color: white;
  font-weight: bold;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.3);
  cursor: pointer;
  z-index: 1000;
}

.toast.sucesso {
  background: #2e7d32;
}

.toast.erro {
  background: #c62828;
}
</style>
