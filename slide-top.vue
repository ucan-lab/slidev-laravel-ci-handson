<template>
  <header
    v-if="$nav.currentLayout !== 'cover'"
    class="absolute top-0 right-0 p-2"
  >
    Laravelプロジェクトで学ぶ！GitHub ActionsによるCIの実践
  </header>
</template>

<style>
header {
  background-color: #F6D247;
  color: #3D3431;
  width: 100%;
  margin-bottom: 20px;
}
</style>
