<script setup>
import { ref, computed, watch, useTemplateRef, onBeforeMount, onBeforeUnmount, onMounted } from 'vue';
import copy from 'copy-text-to-clipboard';
import { exit } from '@tauri-apps/plugin-process';

import emojis from './emoji.json';

const input = useTemplateRef('input');

onMounted(() => {
  input.value.focus();
});

const filter = ref('');
const highlightedIndex = ref(0);
let selectedText = '';

let ctrlIsDown = false;
let shiftIsDown = false;

const filteredEmojis = computed(() => {
  if (!filter.value) return emojis;
  const filterL = filter.value.toLowerCase();
  return emojis.filter(emoji => emoji.name.includes(filterL));
});

watch(
  () => filteredEmojis.value.length,
  (length) => highlightedIndex.value = 0
);

watch(highlightedIndex, (index) => {
  const el = document.getElementById('emoji-'+index);
  if (el) el.scrollIntoView({behavior: 'smooth', block: 'nearest'});
});

function selectEmoji(emoji, index) {
  if (!emoji) {
    if (index > filteredEmojis.value.length - 1) return false;
    emoji = filteredEmojis.value[index];
  }
  if (!emoji) return false;
  highlightedIndex.value = index;
  if (shiftIsDown) {
    selectedText = selectedText + emoji.emoji;
    copy(selectedText);
  } else {
    copy(emoji.emoji);
    // Slight delay seems to have fixed a minor bug where emoji wasn't copied...
    setTimeout(() => exit(0), 10);
  }
}

function emojiNav(back = false) {
  if (back) {
    if (highlightedIndex.value > 0) highlightedIndex.value--;
  } else {
    if (highlightedIndex.value < filteredEmojis.value.length - 1) highlightedIndex.value++;
  }
}

function handleKeydown(event) {
  
  // event.key = "Unidentified" for shift+tab for some reason
  if (event.keyCode === 9 && event.shiftKey) {
    event.preventDefault();
    emojiNav(true);
  }
  
  switch (event.key) {
    case 'Control':
      ctrlIsDown = true;
      break;
    case 'Shift':
      shiftIsDown = true;
      break;
    case 'w':
      if (ctrlIsDown) {
        event.preventDefault();
        exit(0);
      }
      break;
    case 'Escape':
      event.preventDefault();
      filter.value = '';
      input.value.focus();
      break;
    case 'Tab':
      event.preventDefault();
      if (shiftIsDown) emojiNav(true);
      else emojiNav();
      break;
    case 'Enter':
      event.preventDefault();
      selectEmoji(null, highlightedIndex.value);
      break;
  }
  return true;
}

function handleKeyup(event) {
  if (event.key === 'Control') ctrlIsDown = false;
  else if (event.key === 'Shift') shiftIsDown = false;
}

onBeforeMount(() => {
  window.addEventListener('keydown', handleKeydown);
  window.addEventListener('keyup', handleKeyup);
});
onBeforeUnmount(() => {
  window.removeEventListener('keydown', handleKeydown);
  window.removeEventListener('keyup', handleKeyup);
});

</script>

<template>
  <main class="w-full h-full p-4 flex flex-col max-h-full">
    
    <div>
      <input type="text" v-model="filter" ref="input" class="block w-full rounded-md bg-white px-3 py-1.5 text-base text-black outline-1 -outline-offset-1 outline-white/10 placeholder:text-gray-500 focus:outline-2 focus:-outline-offset-2 focus:outline-indigo-500 sm:text-sm/6" />
    </div>
    
    <div class="grow py-2 -mx-1 overflow-y-auto text-center" style="min-height:1px;">
      
      <div v-for="(emoji, index) in filteredEmojis" class="inline-block m-1">
        <button type="button" :id="'emoji-'+index" class="text-3xl cursor-pointer p-1 rounded-md" :class="{ 'bg-gray-300': index == highlightedIndex }" @click="selectEmoji(emoji, index)" :title="emoji.name">
          {{ emoji.emoji }}
        </button>
      </div>
      
    </div>
    
  </main>
</template>
