<template>
  <footer
    v-if="$nav.currentLayout !== 'cover'"
    class="absolute bottom-0 left-0 right-0 flex justify-between items-center"
  >
    <span>{{ $slidev.nav.currentPage }} / {{ $slidev.nav.total }}</span>
    <img src="/miraito.jpg" class="h-10">
  </footer>
</template>

<style>
footer {
  background-color: #E69C37;
  color: #3D3431;
  width: 100%;
  padding-left: 0.5rem;
}
</style>
