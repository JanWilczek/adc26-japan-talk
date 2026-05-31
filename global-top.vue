<!-- hide the footer from "cover" layout -->
<template>
  <footer
    v-if="$nav.currentLayout !== 'cover' && $nav.currentLayout !== 'image'"
  >
    <Scroll/>
  <div class="absolute bottom-5 left-0 right-5 text-right text-xs"><SlideCurrentNo/>/<SlidesTotal/></div>
  </footer>
</template>
