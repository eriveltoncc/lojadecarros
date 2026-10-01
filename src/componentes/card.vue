<template>
  <tr>
    <td>{{ carro.marca }}</td>
    <td>{{ carro.modelo }}</td>
    <td>{{ carro.ano }}</td>
    <td>
      {{
        Number(carro.preco).toLocaleString("pt-BR", {
          style: "currency",
          currency: "BRL",
        })
      }}
    </td>
    <td>
      <button @click="emit('editar', carro.id)">Editar</button>
      <button @click="emit('excluir', carro.id)">Excluir</button>
    </td>
  </tr>
</template>
<script setup>
const props = defineProps({
  carro: {
    type: Object,
    required: true,
  },
});

const emit = defineEmits(["editar", "excluir"]);
</script>

<style scoped>
td {
  padding: 8px;
  border: 1px solid black;
}

button {
  margin-right: 0.5rem;
}
</style>
