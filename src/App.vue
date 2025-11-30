<script setup>
import { ref, reactive, computed, watch, useTemplateRef, onBeforeMount, onBeforeUnmount, onMounted, isProxy, toRaw, nextTick } from 'vue';
import copy from 'copy-text-to-clipboard';
import { exit } from '@tauri-apps/plugin-process';
import { load } from '@tauri-apps/plugin-store';

let store;
let storeLoaded = false;

const emojis = ref([]);
import emojisJson from './emoji.json';
for (let i=0; i<emojisJson.length; ++i) {
  emojisJson[i].id = emojisJson[i].id + '';
  let codes = [];
  for (let j=0; j<emojisJson[i].code.length; ++j) {
    codes.push(parseInt(emojisJson[i].code[j].replace('U+', '0x')));
  }
  emojisJson[i].emoji = String.fromCodePoint(...codes);
  emojis.value.push(emojisJson[i]);
}
console.log('done parsing codes');

const hotEmojis = ref([]);
const filter = ref('');
const highlightedEmojiId = ref('');
const selectedEmojis = ref([]);

let dialogOpen = false;

const settings = reactive({
  theme: 'system',
  emoji_size: 'medium',
  pinned_emojis: [],
  used_emojis: [],
  max_hot_emojis: 10,
  remember_used_emojis: true,
});

const filterInput = useTemplateRef('filterinput');

const dialog = useTemplateRef('dialog');
const dialogBackdrop = useTemplateRef('dialogbackdrop');
const dialogPanel = useTemplateRef('dialogpanel');

const hotEmojisToShow = computed(() => {
  let r = [];
  const maxLength = Math.min(hotEmojis.value.length, settings.max_hot_emojis);
  for (let i=0; i<maxLength; ++i) {
    hotEmojis.value[i].id = 'h-'+(i+1);
    r.push(hotEmojis.value[i]);
  }
  return r;
});

watch(filter, (filterVal) => {
  filterEmojis(filterVal);
  resetHighlightedEmoji();
});

const unfilteredPinnedEmojiIds = computed(() => {
  let ids = [];
  for (let i=0; i<settings.pinned_emojis.length; ++i) {
    if (!settings.pinned_emojis[i].filtered) ids.push(settings.pinned_emojis[i].id);
  }
  return ids;
});

const unfilteredHotEmojiIds = computed(() => {
  let ids = [];
  for (let i=0; i<hotEmojisToShow.value.length; ++i) {
    if (!hotEmojisToShow.value[i].filtered) ids.push(hotEmojisToShow.value[i].id);
  }
  return ids;
});

const unfilteredMainEmojiIds = computed(() => {
  let ids = [];
  for (let i=0; i<emojis.value.length; ++i) {
    if (!emojis.value[i].filtered) ids.push(emojis.value[i].id);
  }
  return ids;
});

const unfilteredEmojiIds = computed(() => {
  return unfilteredPinnedEmojiIds.value.concat(
    unfilteredHotEmojiIds.value,
    unfilteredMainEmojiIds.value
  );
});

watch(highlightedEmojiId, (emojiId) => {
  const el = document.getElementById('emoji-'+emojiId);
  if (el) el.scrollIntoView({behavior: 'smooth', block: 'nearest'});
});

const selectedText = computed(() => {
  let r = '';
  for (let i=0; i<selectedEmojis.value.length; ++i) {
    r = r + selectedEmojis.value[i].emoji;
  }
  return r;
});

const highlightedEmoji = computed(() => {
  for (let i=0; i<settings.pinned_emojis.length; ++i) {
    if (settings.pinned_emojis[i].id === highlightedEmojiId.value)
      return settings.pinned_emojis[i];
  }
  for (let i=0; i<hotEmojisToShow.value.length; ++i) {
    if (hotEmojisToShow.value[i].id === highlightedEmojiId.value)
      return hotEmojisToShow.value[i];
  }
  for (let i=0; i<emojis.value.length; ++i) {
    if (emojis.value[i].id === highlightedEmojiId.value)
      return emojis.value[i];
  }
  return null;
});

watch(selectedText, (text) => copy(text));

watch(
  () => settings.theme,
  (newTheme) => applyTheme(newTheme)
);

watch(settings, (newSettings) => {
  if (isProxy(newSettings)) newSettings = toRaw(newSettings);
  if (store && storeLoaded) store.set('settings', newSettings);
});

function parseIntSafe(val) {
  if (!val) return 0;
  val = parseInt(val);
  if (!val || isNaN(val)) return 0;
  return val;
}

function parseBooleanSafe(val) {
  return !(!val || val === 'false');
}

function emojiMatches(emoji, filter) {
  if (!filter) return true;
  return (
    emoji.name.indexOf(filter) > -1
    || emoji.subcategory.indexOf(filter) > -1
  );
}

function filterEmojis() {
  const filterStr = filter.value.trim().toLowerCase();
  for (let i=0; i<settings.pinned_emojis.length; ++i) {
    settings.pinned_emojis[i].filtered = !emojiMatches(settings.pinned_emojis[i], filterStr);
  }
  for (let i=0; i<hotEmojisToShow.value.length; ++i) {
    hotEmojisToShow.value[i].filtered = !emojiMatches(hotEmojisToShow.value[i], filterStr);
  }
  for (let i=0; i<emojis.value.length; ++i) {
    emojis.value[i].filtered = !emojiMatches(emojis.value[i], filterStr);
  }
}

function applyTheme(theme) {
  document.documentElement.classList.toggle(
    'dark',
    theme === 'dark'
      || (theme === 'system' && window.matchMedia("(prefers-color-scheme: dark)").matches)
  );
}

function copyEmoji(emoji) {
  copy(emoji.emoji);
  addEmojiToHistory(emoji);
  setTimeout(() => exit(0), 10);
}

function addEmojiToSelected(emoji) {
  selectedEmojis.value.push(emoji);
  addEmojiToHistory(emoji);
  highlightedEmojiId.value = emoji.id;
}

function removeSelectedEmoji(index) {
  selectedEmojis.value.splice(index, 1);
}

function pinEmoji(sourceEmoji) {
  let isAlreadyPinned = false;
  // Clone emoji to avoid changing id
  let emoji = {...sourceEmoji};
  let id = sourceEmoji.id.match(/[\d]+$/);
  emoji.id = 'p-'+id;
  for (let i=0; i<settings.pinned_emojis.length; ++i) {
    if (settings.pinned_emojis[i].emoji === emoji.emoji)  {
      settings.pinned_emojis.splice(i, 1);
      settings.pinned_emojis.unshift(emoji);
      isAlreadyPinned = true;
      break;
    }
  }
  if (!isAlreadyPinned) settings.pinned_emojis.unshift(emoji);
  filterEmojis();
  highlightedEmojiId.value = sourceEmoji.id;
}

function unpinEmoji(index) {
  settings.pinned_emojis.splice(index, 1);
  filterEmojis();
}

function emojiNav(forward = true) {
  if (!unfilteredEmojiIds.value.length) {
    highlightedEmojiId.value = '';
    return false;
  }
  let idIndex = unfilteredEmojiIds.value.indexOf(highlightedEmojiId.value);
  if (idIndex < 0) {
    highlightedEmojiId.value = unfilteredEmojiIds.value[0];
  } else {
    if (forward) {
      if (idIndex < unfilteredEmojiIds.value.length - 1) idIndex++;
    } else {
      if (idIndex > 0) idIndex--;
    }
    highlightedEmojiId.value = unfilteredEmojiIds.value[idIndex];
  }
}

function resetHighlightedEmoji() {
  if (unfilteredEmojiIds.value.length) highlightedEmojiId.value = unfilteredEmojiIds.value[0];
  else highlightedEmojiId.value = '';
}

function addEmojiToHistory(emoji) {
  if (parseBooleanSafe(settings.remember_used_emojis))
    settings.used_emojis.unshift(emoji.emoji);
}

function removeEmojiFromHistory(emoji, index) {
  settings.used_emojis = settings.used_emojis.filter(usedEmoji => usedEmoji !== emoji.emoji);
  hotEmojis.value.splice(index, 1);
  filterEmojis();
}

function clearEmojiHistory() {
  settings.used_emojis = [];
  hotEmojis.value = [];
  filterEmojis();
}

function clearFilterInput() {
  filter.value = '';
  filterInput.value.focus();
}

function handleKeydown(event) {
  switch (event.keyCode) {
    case 9: // Tab
      if (!dialogOpen) {
        event.preventDefault();
        emojiNav(!event.shiftKey);
      }
      break;
    case 87: // w
      if (event.ctrlKey) {
        event.preventDefault();
        exit(0);
      }
      break;
    case 27: // Esc
      event.preventDefault();
      if (dialogOpen) closeDialog();
      else clearFilterInput();
      break;
    case 13: // Enter
      event.preventDefault();
      if (!dialogOpen) {
        if (highlightedEmojiId.value) {
          if (event.shiftKey && !event.ctrlKey && !event.altKey) { // Shift
            if (highlightedEmoji.value) addEmojiToSelected(highlightedEmoji.value);
          } else if (event.ctrlKey && !event.shiftKey && !event.altKey) { // Ctrl
            if (/^p-/.test(highlightedEmojiId.value)) { // Pinned
              let index = -1;
              for (let i=0; i<settings.pinned_emojis.length; ++i) {
                if (settings.pinned_emojis[i].id === highlightedEmojiId.value) {
                  index = i;
                  break;
                }
              }
              if (index > -1) unpinEmoji(index);
            } else {
              if (highlightedEmoji.value) pinEmoji(highlightedEmoji.value);
            }
          } else if (event.altKey && !event.shiftKey && !event.ctrlKey) { // Alt
            if (/^h-/.test(highlightedEmojiId.value)) {
              let index = -1;
              for (let i=0; i<hotEmojis.value.length; ++i) {
                if (hotEmojis.value[i].id === highlightedEmojiId.value) {
                  index = i;
                  break;
                }
              }
              if (index > -1 && highlightedEmoji.value) removeEmojiFromHistory(highlightedEmoji.value, index);
            }
          } else if (!event.shiftKey && !event.ctrlKey && !event.altKey) { // No modifiers
            if (highlightedEmoji.value) copyEmoji(highlightedEmoji.value);
          }
        }
      }
      break;
  }
}

function showDialog() {
  dialogOpen = true;
  dialog.value.show();
  setTimeout(() => {
    dialogBackdrop.value.removeAttribute('data-closed');
    dialogPanel.value.removeAttribute('data-closed');
  }, 10);
}

function closeDialog() {
  dialogOpen = false;
  dialogBackdrop.value.setAttribute('data-closed', 'true');
  dialogPanel.value.setAttribute('data-closed', 'true');
  setTimeout(() => {
    dialog.value.close();
  }, 160);
  
}

onBeforeMount(async () => {
  
  store = await load('store.json', { autoSave: true });
  if (store) {
    const storedSettings = await store.get('settings');
    if (storedSettings) {
      if (storedSettings.theme) settings.theme = storedSettings.theme;
      if (storedSettings.pinned_emojis) {
        for (let i=0; i<storedSettings.pinned_emojis.length; ++i) {
          storedSettings.pinned_emojis[i].id = 'p-'+(i+1);
          settings.pinned_emojis.push(storedSettings.pinned_emojis[i]);
        }
      }
      if (storedSettings.max_hot_emojis) settings.max_hot_emojis = parseIntSafe(storedSettings.max_hot_emojis);
      if (storedSettings.emoji_size) settings.emoji_size = storedSettings.emoji_size;
      if (storedSettings.hasOwnProperty('remember_used_emojis')) {
        settings.remember_used_emojis = parseBooleanSafe(storedSettings.remember_used_emojis);
      }
      if (storedSettings.used_emojis) {
        settings.used_emojis = storedSettings.used_emojis;
        // Measure emoji hotness based on frequency & recency of use
        let usedEmojiIndices = [];
        for (let i=0; i<storedSettings.used_emojis.length; ++i) {
          for (let j=0; j<emojis.value.length; ++j) {
            if (storedSettings.used_emojis[i] === emojis.value[j].emoji) {
              if (usedEmojiIndices.indexOf(j) < 0) usedEmojiIndices.push(j);
              emojis.value[j].hotness = (emojis.value[j].hotness ?? 0) + (storedSettings.used_emojis.length - i);
              break;
            }
          }
        }
        usedEmojiIndices.sort((a, b) => emojis.value[b].hotness - emojis.value[a].hotness);
        for (let i=0; i<usedEmojiIndices.length; ++i) {
          hotEmojis.value.push({...emojis.value[usedEmojiIndices[i]]});
        }
      }
    }
    nextTick(() => {
      filterEmojis();
      resetHighlightedEmoji();
      storeLoaded = true;
    });
  }
  document.body.removeAttribute('style');
  
  window.addEventListener('keydown', handleKeydown);
});

onBeforeUnmount(() => {
  window.removeEventListener('keydown', handleKeydown);
});

onMounted(() => filterInput.value.focus());

</script>

<template>
  <main class="flex flex-col w-full h-full max-h-full">
    
    <div class="px-4 py-2 bg-gray-200 shrink-0 dark:bg-gray-900">
      <div class="relative">
        <input type="text" v-model="filter" placeholder="Search" ref="filterinput" id="filterinput" class="block w-full rounded-md bg-white dark:bg-gray-800 px-3 py-1.5 text-base text-black dark:text-gray-100 outline-1 -outline-offset-1 outline-white/10 placeholder:text-gray-500 focus:outline-0 focus-visible:-outline-offset-2 focus-visible:outline-indigo-500 sm:text-sm/6" />
        <button type="button" @click="clearFilterInput" class="absolute top-0 right-0 flex items-center justify-center text-gray-700 rounded-md cursor-pointer size-10 dark:text-gray-300">
          <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-x-icon lucide-x size-5"><path d="M18 6 6 18"/><path d="m6 6 12 12"/></svg>
        </button>
      </div>
    </div>
    
    <div class="flex flex-wrap items-start justify-center gap-2 py-2 overflow-y-auto grow" style="min-height:1px;">
      
      <button v-for="(emoji, index) in settings.pinned_emojis"
        type="button"
        v-show="!emoji.filtered"
        :id="'emoji-'+emoji.id"
        :title="emoji.name"
        class="relative flex items-center justify-center rounded-md cursor-pointer focus:outline-none min-w-8 min-h-8"
        :class="{
          'text-lg': settings.emoji_size === 'small',
          'text-3xl': settings.emoji_size === 'medium',
          'text-6xl': settings.emoji_size === 'large',
          'bg-gray-300 dark:bg-gray-600 ring ring-gray-500': emoji.id === highlightedEmojiId
        }"
        @click.exact="copyEmoji(emoji)"
        @click.shift.exact="addEmojiToSelected(emoji, index)"
        @click.ctrl.exact="unpinEmoji(index)"
        tabindex="-1"
        >
        <div class="absolute top-0 flex items-center justify-center text-yellow-400 rounded-full size-4 -right-1">
          <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="currentColor" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-star-icon lucide-star size-3"><path d="M11.525 2.295a.53.53 0 0 1 .95 0l2.31 4.679a2.123 2.123 0 0 0 1.595 1.16l5.166.756a.53.53 0 0 1 .294.904l-3.736 3.638a2.123 2.123 0 0 0-.611 1.878l.882 5.14a.53.53 0 0 1-.771.56l-4.618-2.428a2.122 2.122 0 0 0-1.973 0L6.396 21.01a.53.53 0 0 1-.77-.56l.881-5.139a2.122 2.122 0 0 0-.611-1.879L2.16 9.795a.53.53 0 0 1 .294-.906l5.165-.755a2.122 2.122 0 0 0 1.597-1.16z"/></svg>
        </div>
        {{ emoji.emoji }}
      </button>
      
      <button v-for="(emoji, index) in hotEmojisToShow"
        type="button"
        v-show="!emoji.filtered"
        :id="'emoji-'+emoji.id"
        :title="emoji.name"
        class="relative flex items-center justify-center rounded-md cursor-pointer focus:outline-none min-w-8 min-h-8"
        :class="{
          'text-lg': settings.emoji_size === 'small',
          'text-3xl': settings.emoji_size === 'medium',
          'text-6xl': settings.emoji_size === 'large',
          'bg-gray-300 dark:bg-gray-600 ring ring-gray-500': emoji.id === highlightedEmojiId
        }"
        @click.exact="copyEmoji(emoji)"
        @click.shift.exact="addEmojiToSelected(emoji, index)"
        @click.ctrl.exact="pinEmoji(emoji)"
        @click.alt.exact="removeEmojiFromHistory(emoji, index)"
        tabindex="-1"
        >
        <div class="absolute top-0 flex items-center justify-center text-gray-500 rounded-full size-4 dark:text-gray-300 -right-1">
          <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-clock-icon lucide-clock size-3"><path d="M12 6v6l4 2"/><circle cx="12" cy="12" r="10"/></svg>
        </div>
        {{ emoji.emoji }}
      </button>
      
      <button v-for="(emoji, index) in emojis"
        type="button"
        v-show="!emoji.filtered"
        :id="'emoji-'+emoji.id"
        :title="emoji.name"
        class="flex items-center justify-center rounded-md cursor-pointer focus:outline-none min-w-8 min-h-8"
        :class="{
          'text-lg': settings.emoji_size === 'small',
          'text-3xl': settings.emoji_size === 'medium',
          'text-6xl': settings.emoji_size === 'large',
          'bg-gray-300 dark:bg-gray-600 ring ring-gray-500': emoji.id === highlightedEmojiId
        }"
        @click.exact="copyEmoji(emoji)"
        @click.shift.exact="addEmojiToSelected(emoji, index)"
        @click.ctrl.exact="pinEmoji(emoji)"
        tabindex="-1"
        >
        {{ emoji.emoji }}
      </button>
      
    </div>
    
    <div class="flex items-center gap-8 px-4 py-2 bg-gray-200 shrink-0 dark:bg-gray-800">
      <div class="flex flex-wrap items-center gap-1 grow">
        <div class="flex items-center justify-center text-gray-400 size-10">
          <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-clipboard-icon lucide-clipboard"><rect width="8" height="4" x="8" y="2" rx="1" ry="1"/><path d="M16 4h2a2 2 0 0 1 2 2v14a2 2 0 0 1-2 2H6a2 2 0 0 1-2-2V6a2 2 0 0 1 2-2h2"/></svg>
        </div>
        <div v-for="(emoji, index) in selectedEmojis">
          <button type="button"
            class="relative flex items-center justify-center text-3xl rounded-md cursor-pointer focus:outline-none min-w-8 min-h-8 group"
            @click="removeSelectedEmoji(index)"
            :title="'Remove ' + emoji.name + ' from clipboard'"
            >
            <div class="absolute top-0 right-0 flex items-center justify-center text-gray-900 transition-opacity rounded-full opacity-0 size-4 bg-gray-600/40 dark:bg-gray-200/40 dark:text-gray-50 group-hover:opacity-100">
              <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-x-icon lucide-x size-3"><path d="M18 6 6 18"/><path d="m6 6 12 12"/></svg>
            </div>
            {{ emoji.emoji }}
          </button>
        </div>
      </div>
      <button type="button" class="flex items-center justify-center text-blue-800 rounded-md cursor-pointer shrink-0 size-8 dark:text-blue-100 focus:outline-none" @click="showDialog">
        <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-settings-icon lucide-settings size-5"><path d="M9.671 4.136a2.34 2.34 0 0 1 4.659 0 2.34 2.34 0 0 0 3.319 1.915 2.34 2.34 0 0 1 2.33 4.033 2.34 2.34 0 0 0 0 3.831 2.34 2.34 0 0 1-2.33 4.033 2.34 2.34 0 0 0-3.319 1.915 2.34 2.34 0 0 1-4.659 0 2.34 2.34 0 0 0-3.32-1.915 2.34 2.34 0 0 1-2.33-4.033 2.34 2.34 0 0 0 0-3.831A2.34 2.34 0 0 1 6.35 6.051a2.34 2.34 0 0 0 3.319-1.915"/><circle cx="12" cy="12" r="3"/></svg>
      </button>
    </div>
    
    <dialog ref="dialog" id="dialog" aria-labelledby="dialog-title" class="fixed inset-0 overflow-y-auto bg-transparent size-auto max-h-none max-w-none">
      
      <div data-closed ref="dialogbackdrop" class="fixed inset-0 transition-opacity duration-150 bg-gray-800/75 data-closed:opacity-0" @click="closeDialog"></div>
      
      <div tabindex="0" class="flex items-center justify-center h-full max-h-screen min-h-full p-4 text-center focus:outline-none sm:p-0">
        
        <div data-closed ref="dialogpanel" class="relative h-auto max-h-screen text-left transition-all duration-150 transform bg-white rounded-lg shadow-xl dark:bg-black dark:text-gray-50 data-closed:translate-y-4 data-closed:opacity-0 sm:my-8 sm:w-full sm:max-w-lg data-closed:sm:translate-y-0 data-closed:sm:scale-95">
          
          <div class="flex flex-col max-h-screen pt-4 pb-6 sm:pb-8 sm:pt-4">
            
            <div class="flex items-center justify-between px-4 shrink-0 gap-x-8 sm:px-6">
              <h3 class="text-2xl font-medium text-gray-600 dark:text-gray-300">
                Settings
              </h3>
              <button type="button" @click="closeDialog" class="relative flex items-center justify-center rounded-md cursor-pointer size-10 -right-2">
                <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-x-icon lucide-x size-5"><path d="M18 6 6 18"/><path d="m6 6 12 12"/></svg>
              </button>
            </div>
            
            <div class="px-4 overflow-y-auto sm:px-6" style="min-height:1px;">
              
              <div class="flex flex-wrap items-start gap-8 mt-3">
                
                <div>
                  <label for="theme" class="block text-sm font-semibold text-gray-700 dark:text-gray-200">Theme</label>
                  <select v-model="settings.theme" id="theme" class="mt-1 rounded-md bg-white dark:bg-gray-800 pl-3 py-1.5 text-base text-black dark:text-gray-100 outline-1 -outline-offset-1 outline-white/10 focus:outline-0 focus-visible:-outline-offset-2 focus-visible:outline-indigo-500 sm:text-sm/6">
                    <option value="system">System</option>
                    <option value="light">Light</option>
                    <option value="dark">Dark</option>
                  </select>
                </div>
                
                <div>
                  <label for="emoji-size" class="block text-sm font-semibold text-gray-700 dark:text-gray-200">Emoji Size</label>
                  <select v-model="settings.emoji_size" id="emoji-size" class="mt-1 rounded-md bg-white dark:bg-gray-800 pl-3 py-1.5 text-base text-black dark:text-gray-100 outline-1 -outline-offset-1 outline-white/10 focus:outline-0 focus-visible:-outline-offset-2 focus-visible:outline-indigo-500 sm:text-sm/6">
                    <option value="small">Small</option>
                    <option value="medium">Medium</option>
                    <option value="large">Large</option>
                  </select>
                </div>
                
                <div>
                  <label for="max-hot-emojis" class="block text-sm font-semibold text-gray-700 dark:text-gray-200">Max History</label>
                  <input type="number" id="max-hot-emojis" min="0" max="50" v-model="settings.max_hot_emojis" class="mt-1 w-24 rounded-md bg-white dark:bg-gray-800 px-3 py-1.5 text-base text-black dark:text-gray-100 outline-1 -outline-offset-1 outline-white/10 placeholder:text-gray-500 focus:outline-0 focus-visible:-outline-offset-2 focus-visible:outline-indigo-500 sm:text-sm/6" />
                </div>
                
              </div>
              
              <div class="flex flex-wrap items-center gap-8 mt-8">
                
                <button type="button" @click="clearEmojiHistory" class="flex items-center justify-center h-10 gap-2 px-4 text-sm font-semibold bg-gray-200 rounded-md cursor-pointer darK:text-gray-100 dark:bg-gray-700 shrink-0 whitespace-nowrap">
                  <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-clock-icon lucide-clock size-4"><path d="M12 6v6l4 2"/><circle cx="12" cy="12" r="10"/></svg>
                  Clear history
                </button>
                
                <label for="remember_used_emojis" class="flex items-center gap-2 cursor-pointer">
                  <input id="remember_used_emojis" type="checkbox" v-model="settings.remember_used_emojis" class="rounded-md" />
                  Remember used emojis
                </label>
          
              </div>
              
              <h3 class="mt-8 text-2xl font-medium text-gray-600 dark:text-gray-300">
                Controls
              </h3>
              
              <p class="mt-3 text-sm">
                Click or <span class="font-semibold">Enter</span> to copy an emoji and close the app.
              </p>
              <p class="mt-3 text-sm">
                Hold <span class="font-semibold">Shift</span> to copy multiple emojis. Copied emojis will be displayed at the bottom of the screen. Click any copied emoji from this list to remove it from your clipboard.
              </p>
              <p class="mt-3 text-sm">
                <span class="font-semibold">Tab</span> / <span class="font-semibold">Shift</span> + <span class="font-semibold">Tab</span> to navigate through emojis.
              </p>
              <p class="mt-3 text-sm">
                Hold <span class="font-semibold">Ctrl</span> with click/Enter to add the currently highlighted emoji to your favourites. <span class="font-semibold">Ctrl</span> + click/Enter to remove an emoji from favourites.
              </p>
              <p class="mt-3 text-sm">
                Hold <span class="font-semibold">Alt</span> with click/Enter to remove the currently highlighted emoji from your history.
              </p>
              <p class="mt-3 text-sm">
                <span class="font-semibold">Esc</span> to clear the search box (or close this dialog).
              </p>
              <p class="mt-3 text-sm">
                <span class="font-semibold">Ctrl</span> + <span class="font-semibold">W</span> to close the app.
              </p>
              
            </div>
            
          </div>
          
        </div>
        
      </div>
      
    </dialog>
    
  </main>
</template>
