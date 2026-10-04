<template>
  <div class="slide-number" aria-label="Номер слайда">{{ $page }} / {{ $nav.total }}</div>
</template>

<style scoped>
.slide-number {
  position: absolute;
  right: 1.25rem;
  bottom: 0.8rem;
  padding: 0.1rem 0.4rem;
  color: var(--talk-text);
  background: #0b0d0fcc;
  font: 0.9rem ui-monospace, SFMono-Regular, Menlo, monospace;
  pointer-events: none;
}
</style>
