<script setup>
import { ref, reactive, computed, watch, useTemplateRef, onBeforeMount, onBeforeUnmount, onMounted, isProxy, toRaw, nextTick } from 'vue';
import copy from 'copy-text-to-clipboard';
import { exit } from '@tauri-apps/plugin-process';
import { load } from '@tauri-apps/plugin-store';
// import { getVersion } from '@tauri-apps/api/app';

/**
 * Variables
 */

// Reactive
const emojis = ref({});
const hotEmojis = ref([]);
const filter = ref('');
const highlightedEmojiIndex = ref(0);
const selectedEmojis = ref([]);

// Not reactive
const appVersion = '1.0.0';
let emojiNames = [];
let store = null;
let storeLoaded = false;
let dialogOpen = false;

// Element refs
const filterInput = useTemplateRef('filterinput');
const dialog = useTemplateRef('dialog');
const dialogBackdrop = useTemplateRef('dialogbackdrop');
const dialogPanel = useTemplateRef('dialogpanel');

/**
 * Setup
 */

import emojisJson from './emojis-with-modifiers.json';

const skinToneKeys = ['lst','mlst','mst','mdst','dst'];

for (let i=0; i<emojisJson.length; ++i) {
  emojisJson[i].searchable_text = (emojisJson[i].name + ' ' + emojisJson[i].category + ' ' + emojisJson[i].category).toLowerCase();
  emojisJson[i].filtered = false;
  emojisJson[i].supported = true;
  let altIndices = {};
  if (emojisJson[i].alts.length) {
    for (let j=0; j<emojisJson[i].alts.length; ++j) {
      emojisJson[i].alts[j].supported = true;
    }
    for (const stk1 of skinToneKeys) {
      stk2Loop: for (const stk2 of skinToneKeys) {
        for (let j=0; j<emojisJson[i].alts.length; ++j) {
          if (emojisJson[i].alts[j].st1 === stk1 && emojisJson[i].alts[j].st2 === stk2) {
            altIndices[stk1+'.'+stk2] = j;
            continue stk2Loop;
          }
        }
        for (let j=0; j<emojisJson[i].alts.length; ++j) {
          if (emojisJson[i].alts[j].st1 === stk1 && emojisJson[i].alts[j].st2 === null) {
            altIndices[stk1+'.'+stk2] = j;
            continue stk2Loop;
          }
        }
      }
    }
    emojisJson[i].alt_indices = altIndices;
  }
  emojis.value[emojisJson[i].name] = emojisJson[i];
  emojiNames.push(emojisJson[i].name);
}

let theme = getAttrFromLocalStorage('theme', 'system');
if (theme !== 'light' && theme !== 'dark') theme = 'system';
let emojiSize = getAttrFromLocalStorage('emoji_size', 'medium');
if (emojiSize !== 'small' && emojiSize !== 'large') emojiSize = 'medium';
let maxHotEmojis = getAttrFromLocalStorage('max_hot_emojis', 10);
maxHotEmojis = parseInt(maxHotEmojis);
if (isNaN(maxHotEmojis)) maxHotEmojis = 10;
let showUnsupportedEmojis = getAttrFromLocalStorage('show_unsupported_emojis', false);
if (showUnsupportedEmojis !== true && showUnsupportedEmojis !== 'true') showUnsupportedEmojis = false;

const settings = reactive({
  theme: theme,
  emoji_size: emojiSize,
  pinned_emojis: [],
  used_emojis: [],
  max_hot_emojis: maxHotEmojis,
  remember_used_emojis: true,
  show_unsupported_emojis: false,
  st1: null,
  st2: null,
});

onBeforeMount(async () => {
  
  store = await load('store.json', { autoSave: true });
  if (store) {
    const storedSettings = await store.get('settings');
    if (storedSettings) {
      if (storedSettings.theme) settings.theme = storedSettings.theme;
      if (storedSettings.pinned_emojis) settings.pinned_emojis = storedSettings.pinned_emojis;
      if (storedSettings.max_hot_emojis) settings.max_hot_emojis = parseIntSafe(storedSettings.max_hot_emojis);
      if (storedSettings.emoji_size) settings.emoji_size = storedSettings.emoji_size;
      if (storedSettings.st1) settings.st1 = storedSettings.st1;
      if (storedSettings.st2) settings.st2 = storedSettings.st2;
      if (storedSettings.hasOwnProperty('remember_used_emojis')) {
        settings.remember_used_emojis = parseBooleanSafe(storedSettings.remember_used_emojis);
      }
      if (storedSettings.used_emojis) {
        settings.used_emojis = storedSettings.used_emojis.slice(0, 100);
        // Measure emoji hotness based on frequency & recency of use
        let usedEmojiData = [];
        usedEmojiLoop: for (let i=0; i<storedSettings.used_emojis.length; ++i) {
          for (let j=0; j<usedEmojiData.length; ++j) {
            if (emojiDataEquals(storedSettings.used_emojis[i], usedEmojiData[j])) {
              usedEmojiData[j].hotness += storedSettings.used_emojis.length - i;
              continue usedEmojiLoop;
            }
          }
          usedEmojiData.push({
            name: storedSettings.used_emojis[i].name,
            modifier: storedSettings.used_emojis[i].modifier,
            hotness: storedSettings.used_emojis.length - i,
          });
        }
        usedEmojiData.sort((a, b) => b.hotness - a.hotness);
        hotEmojis.value = usedEmojiData;
      }
    }
    nextTick(() => storeLoaded = true);
  }
  
  // appVersion.value = await getVersion();
      
  setTimeout(() => {
    // Set up comparison element for supported emojis
    const compareEl = document.createElement('span');
    compareEl.style = "position:absolute;visibility:hidden;";
    compareEl.textContent = '☺️';
    document.body.appendChild(compareEl);
    const compareSize = compareEl.getBoundingClientRect();
    const compareW = compareSize.width;
    const compareH = compareSize.height;
    compareEl.remove();
    const el = document.createElement('span');
    el.style = "position:absolute;visibility:hidden;";
    el.textContent = '☺️';
    document.body.appendChild(el);
    let elSize;
    let codes;
    let emoji;
    let supported;
    for (let i=0; i<emojiNames.length; ++i) {
      emoji = unwrap(emojis.value[emojiNames[i]]);
      el.textContent = emoji.emoji;
      elSize = el.getBoundingClientRect();
      supported = (elSize.width === compareW && elSize.height === compareH);
      if (!supported) {
        // Try recreating emoji from codes in case that works
        codes = [];
        for (let j=0; j<emoji.code.length; ++j) {
          codes.push(parseInt(emoji.code[j].replace('U+', '0x')));
        }
        emojis.value[emojiNames[i]].emoji = String.fromCodePoint(...codes);
        el.textContent = emojis.value[emojiNames[i]].emoji;
        elSize = el.getBoundingClientRect();
        supported = (elSize.width === compareW && elSize.height === compareH);
      } else {
        supported = true;
      }
      emojis.value[emojiNames[i]].supported = supported;
      for (let k=0; k<emoji.alts.length; ++k) {
        el.textContent = emoji.alts[k].emoji;
        elSize = el.getBoundingClientRect();
        supported = (elSize.width === compareW && elSize.height === compareH);
        if (!supported) {
          // Try recreating emoji from codes in case that works
          codes = [];
          for (let j=0; j<emoji.alts[k].code.length; ++j) {
            codes.push(parseInt(emoji.alts[k].code[j].replace('U+', '0x')));
          }
          emojis.value[emojiNames[i]].alts[k].emoji = String.fromCodePoint(...codes);
          el.textContent = emojis.value[emojiNames[i]].alts[k].emoji;
          elSize = el.getBoundingClientRect();
          supported = (elSize.width === compareW && elSize.height === compareH);
        } else {
          supported = true;
        }
        emojis.value[emojiNames[i]].alts[k].supported = supported;
      }
    }
    el.remove();
  }, 10);

  window.addEventListener('keydown', handleKeydown);
});

onBeforeUnmount(() => {
  window.removeEventListener('keydown', handleKeydown);
});

onMounted(() => filterInput.value.focus());

/**
 * Computed
 */

const unfilteredEmojis = computed(() => {
  let r = [];
  let emoji;
  const modifier = settings.st1 ? settings.st1+'.'+(settings.st2 ?? settings.st1) : false;
  for (let i=0; i<emojiNames.length; ++i) {
    emoji = emojis.value[emojiNames[i]];
    if (emoji.filtered) continue;
    if (modifier && emoji.alts.length) {
      if (settings.show_unsupported_emojis) {
        r.push({ name: emojiNames[i], modifier: modifier, pinned: false, hot: false });
      } else {
        if (emoji.alts[emoji.alt_indices[modifier]].supported) {
          r.push({ name: emojiNames[i], modifier: modifier, pinned: false, hot: false });
        } else if (emoji.supported) { // Use base emoji if supported
          r.push({ name: emojiNames[i], modifier: false, pinned: false, hot: false });
        }
      }
    } else if (!(!emoji.supported && !settings.show_unsupported_emojis)) {
      r.push({ name: emojiNames[i], modifier: false, pinned: false, hot: false });
    }
  }
  return r;
});

const unfilteredPinnedEmojis = computed(() => {
  let r = [];
  let emoji;
  for (let i=0; i<settings.pinned_emojis.length; ++i) {
    emoji = emojis.value[settings.pinned_emojis[i].name];
    if (!emoji || emoji.filtered) continue;
    if (settings.pinned_emojis[i].modifier) {
      if (!(
        !emoji.alts[emoji.alt_indices[settings.pinned_emojis[i].modifier]].supported
        && !settings.show_unsupported_emojis
      )) {
        r.push({ name: emoji.name, modifier: settings.pinned_emojis[i].modifier, pinned: true, hot: false });
      }
    } else {
      if (!(!emoji.supported && !settings.show_unsupported_emojis)) {
        r.push({ name: emoji.name, modifier: settings.pinned_emojis[i].modifier, pinned: true, hot: false });
      }
    }
  }
  return r;
});

const maxHotEmojisInt = computed(() => parseIntSafe(settings.max_hot_emojis));

const unfilteredHotEmojis = computed(() => {
  let r = [];
  let emoji;
  const maxLength = Math.min(hotEmojis.value.length, maxHotEmojisInt.value);
  for (let i=0; i<maxLength; ++i) {
    emoji = emojis.value[hotEmojis.value[i].name];
    if (!emoji || emoji.filtered) continue;
    if (hotEmojis.value[i].modifier) {
      if (!(
        !emoji.alts[emoji.alt_indices[hotEmojis.value[i].modifier]].supported
        && !settings.show_unsupported_emojis
      )) {
        r.push({ name: emoji.name, modifier: hotEmojis.value[i].modifier, pinned: false, hot: true });
      }
    } else {
      if (!(!emoji.supported && !settings.show_unsupported_emojis)) {
        r.push({ name: emoji.name, modifier: hotEmojis.value[i].modifier, pinned: false, hot: true });
      }
    }
  }
  return r;
});

const allUnfilteredEmojis = computed(() => {
  return unfilteredPinnedEmojis.value.concat(
    unfilteredHotEmojis.value,
    unfilteredEmojis.value
  );
});

const selectedText = computed(() => {
  let r = '';
  let emoji;
  for (let i=0; i<selectedEmojis.value.length; ++i) {
    emoji = unwrap(emojis.value[selectedEmojis.value[i].name]);
    if (selectedEmojis.value[i].modifier) {
      r = r + emoji.alts[emoji.alt_indices[selectedEmojis.value[i].modifier]].emoji;
    } else {
      r = r + emoji.emoji;
    }
  }
  return r;
});

/**
 * Watchers
 */

watch(filter, (filterVal) => {
  filterEmojis();
  nextTick(() => highlightedEmojiIndex.value = 0);
});

watch(highlightedEmojiIndex, (emojiIndex) => {
  const el = document.getElementById('emoji-'+emojiIndex);
  if (el) el.scrollIntoView({behavior: 'smooth', block: 'nearest'});
});

watch(selectedText, (text) => copy(text));

watch(
  () => settings.theme,
  (newTheme) => applyTheme(newTheme)
);

watch(
  () => settings.st1,
  (newSt1) => {
    if (newSt1) {
      if (!settings.st2) settings.st2 = newSt1
    } else if (settings.st2) {
      settings.st2 = null;
    }
  }
);

watch(settings, (newSettings) => {
  newSettings = unwrap(newSettings);
  if (store && storeLoaded) store.set('settings', newSettings);
  // Save some values to storage for faster loading
  localStorage.theme = newSettings.theme;
  localStorage.emoji_size = newSettings.emoji_size;
  localStorage.max_hot_emojis = newSettings.max_hot_emojis;
  localStorage.show_unsupported_emojis = newSettings.show_unsupported_emojis;
  localStorage.st1 = newSettings.st1;
  localStorage.st2 = newSettings.st2;
});

/**
 * Functions
 */

function parseIntSafe(val) {
  if (!val) return 0;
  val = parseInt(val);
  if (!val || isNaN(val)) return 0;
  return val;
}

function parseBooleanSafe(val) {
  return !(!val || val === 'false');
}

function unwrap(val) {
  if (isProxy(val)) return toRaw(val);
  return val;
}
 
function getAttrFromLocalStorage(attrName, defaultVal) {
  if (attrName in localStorage) return localStorage[attrName];
  return defaultVal;
}

function emojiDataEquals(emojiData1, emojiData2) {
  return emojiData1.name === emojiData2.name && emojiData1.modifier === emojiData2.modifier;
}

function filterEmojis() {
  const filterStr = filter.value.trim().toLowerCase();
  for (let i=0; i<emojiNames.length; ++i) {
    emojis.value[emojiNames[i]].filtered = (
      !!filterStr
      && emojis.value[emojiNames[i]].searchable_text.indexOf(filterStr) < 0
    );
  }
}

function emojiNav(forward = true, steps = 1) {
  if (!allUnfilteredEmojis.value.length) {
    highlightedEmojiIndex.value = 0;
    return false;
  }
  if (forward) {
    highlightedEmojiIndex.value += Math.min(steps, allUnfilteredEmojis.value.length - 1 - highlightedEmojiIndex.value);
  } else {
    highlightedEmojiIndex.value -= Math.min(steps, highlightedEmojiIndex.value);
  }
}

function applyTheme(theme) {
  document.documentElement.classList.toggle(
    'dark',
    theme === 'dark'
      || (theme === 'system' && window.matchMedia("(prefers-color-scheme: dark)").matches)
  );
}

function copyEmoji(emojiData) {
  const emoji = unwrap(emojis.value[emojiData.name]);
  if (emojiData.modifier) {
    copy(emoji.alts[emoji.alt_indices[emojiData.modifier]].emoji);
  } else {
    copy(emoji.emoji);
  }
  addEmojiToHistory(emojiData);
  setTimeout(() => exit(0), 10);
}

function addEmojiToSelected(emojiData, emojiIndex) {
  selectedEmojis.value.push({ name: emojiData.name, modifier: emojiData.modifier });
  addEmojiToHistory(emojiData);
  highlightedEmojiIndex.value = emojiIndex;
}

function removeSelectedEmoji(index) {
  selectedEmojis.value.splice(index, 1);
}

function pinEmoji(emojiData, emojiIndex) {
  let wasAlreadyPinned = false;
  for (let i=0; i<settings.pinned_emojis.length; ++i) {
    if (emojiDataEquals(settings.pinned_emojis[i], emojiData)) {
      settings.pinned_emojis.splice(i, 1);
      wasAlreadyPinned = true;
      break;
    }
  }
  if (!wasAlreadyPinned) settings.pinned_emojis.unshift({ name: emojiData.name, modifier: emojiData.modifier });
  highlightedEmojiIndex.value = emojiIndex;
  if (!emojiData.pinned) emojiNav(!wasAlreadyPinned);
}

function addEmojiToHistory(emojiData) {
  if (parseBooleanSafe(settings.remember_used_emojis)) {
    settings.used_emojis.unshift({ name: emojiData.name, modifier: emojiData.modifier });
  }
}

function removeEmojiFromHistory(emojiData, emojiIndex) {
  hotEmojis.value = hotEmojis.value.filter(hotEmoji => !emojiDataEquals(hotEmoji, emojiData));
  settings.used_emojis = settings.used_emojis.filter(usedEmoji => !emojiDataEquals(usedEmoji, emojiData));
  highlightedEmojiIndex.value = emojiIndex;
  nextTick(() => {
    if (highlightedEmojiIndex.value > allUnfilteredEmojis.value.length) {
      highlightedEmojiIndex.value = Math.max(0, allUnfilteredEmojis.value.length - 1);
    }
  });
}

function clearEmojiHistory() {
  hotEmojis.value = [];
  settings.used_emojis = [];
  nextTick(() => {
    if (highlightedEmojiIndex.value > allUnfilteredEmojis.value.length) {
      highlightedEmojiIndex.value = Math.max(0, allUnfilteredEmojis.value.length - 1);
    }
  });
}

function clearFilterInput() {
  filter.value = '';
  filterInput.value.focus();
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
        if (highlightedEmojiIndex.value > allUnfilteredEmojis.value.length - 1) return false;
        const emojiData = allUnfilteredEmojis.value[highlightedEmojiIndex.value];
        if (!event.shiftKey && !event.ctrlKey && !event.altKey) {
          copyEmoji(emojiData);
        } else if (event.shiftKey && !event.ctrlKey && !event.altKey) {
          addEmojiToSelected(emojiData, highlightedEmojiIndex.value);
        } else if (!event.shiftKey && event.ctrlKey && !event.altKey) {
          pinEmoji(emojiData, highlightedEmojiIndex.value);
          console.log(emojiData);
        } else if (!event.shiftKey && !event.ctrlKey && event.altKey) {
          removeEmojiFromHistory(emojiData, highlightedEmojiIndex.value);
        }
      }
      break;
  }
}

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
    
    <div class="overflow-y-auto grow" style="min-height:1px;">
      <div class="flex flex-wrap items-start justify-center gap-2 p-2">
        
        <button type="button"
          v-for="(emojiData, emojiIndex) in allUnfilteredEmojis"
          :id="'emoji-'+emojiIndex"
          :title="emojis[emojiData.name].name"
          class="relative flex items-center justify-center p-1 rounded-md cursor-pointer focus:outline-none min-w-10 min-h-10"
          :class="{
            'text-lg': settings.emoji_size === 'small',
            'text-3xl': settings.emoji_size === 'medium',
            'text-6xl': settings.emoji_size === 'large',
            'bg-gray-300': emojiIndex === highlightedEmojiIndex
          }"
          @click.exact="copyEmoji(emojiData)"
          @click.shift.exact="addEmojiToSelected(emojiData, emojiIndex)"
          @click.ctrl.exact="pinEmoji(emojiData, emojiIndex)"
          @click.alt.exact="removeEmojiFromHistory(emojiData, emojiIndex)"
          >
          
          <div v-if="emojiData.pinned" class="absolute top-0 flex items-center justify-center text-yellow-400 rounded-full size-4 -right-1">
            <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="currentColor" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-star-icon lucide-star size-3"><path d="M11.525 2.295a.53.53 0 0 1 .95 0l2.31 4.679a2.123 2.123 0 0 0 1.595 1.16l5.166.756a.53.53 0 0 1 .294.904l-3.736 3.638a2.123 2.123 0 0 0-.611 1.878l.882 5.14a.53.53 0 0 1-.771.56l-4.618-2.428a2.122 2.122 0 0 0-1.973 0L6.396 21.01a.53.53 0 0 1-.77-.56l.881-5.139a2.122 2.122 0 0 0-.611-1.879L2.16 9.795a.53.53 0 0 1 .294-.906l5.165-.755a2.122 2.122 0 0 0 1.597-1.16z"/></svg>
          </div>
          
          <div v-if="emojiData.hot" class="absolute top-0 flex items-center justify-center text-gray-500 rounded-full size-4 dark:text-gray-300 -right-1">
            <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-clock-icon lucide-clock size-3"><path d="M12 6v6l4 2"/><circle cx="12" cy="12" r="10"/></svg>
          </div>
          
          <span v-if="emojiData.modifier" class="leading-none">
            {{ emojis[emojiData.name].alts[emojis[emojiData.name].alt_indices[emojiData.modifier]].emoji }}
          </span>
          <span v-else class="leading-none">
            {{ emojis[emojiData.name].emoji }}
          </span>
          
        </button>
        
      </div>
    </div>
    
    <div class="flex items-center px-4 py-2 bg-gray-200 shrink-0 dark:bg-gray-800">
      <div class="flex items-center justify-center text-gray-400 shrink-0 size-10">
        <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-clipboard-icon lucide-clipboard"><rect width="8" height="4" x="8" y="2" rx="1" ry="1"/><path d="M16 4h2a2 2 0 0 1 2 2v14a2 2 0 0 1-2 2H6a2 2 0 0 1-2-2V6a2 2 0 0 1 2-2h2"/></svg>
      </div>
      <div class="flex flex-wrap items-center gap-1 grow">
        <div v-for="(emojiData, index) in selectedEmojis">
          <button type="button"
            class="flex items-center justify-center p-1 text-3xl rounded-md cursor-pointer group focus:outline-none min-w-10 min-h-10"
            @click="removeSelectedEmoji(index)"
            :title="'Remove ' + emojiData.name + ' from clipboard'"
            >
            <div class="absolute top-0 right-0 flex items-center justify-center text-gray-900 transition-opacity rounded-full opacity-0 size-4 bg-gray-600/40 dark:bg-gray-200/40 dark:text-gray-50 group-hover:opacity-100">
              <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-x-icon lucide-x size-3"><path d="M18 6 6 18"/><path d="m6 6 12 12"/></svg>
            </div>
            <span v-if="emojiData.modifier" class="leading-none">
            {{ emojis[emojiData.name].alts[emojis[emojiData.name].alt_indices[emojiData.modifier]].emoji }}
          </span>
          <span v-else class="leading-none">
            {{ emojis[emojiData.name].emoji }}
          </span>
          </button>
        </div>
      </div>
      <button type="button" class="flex items-center justify-center p-1 ml-8 text-blue-800 rounded-md cursor-pointer shrink-0 size-10 dark:text-blue-100 focus:outline-none" @click="showDialog">
        <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-settings-icon lucide-settings size-5"><path d="M9.671 4.136a2.34 2.34 0 0 1 4.659 0 2.34 2.34 0 0 0 3.319 1.915 2.34 2.34 0 0 1 2.33 4.033 2.34 2.34 0 0 0 0 3.831 2.34 2.34 0 0 1-2.33 4.033 2.34 2.34 0 0 0-3.319 1.915 2.34 2.34 0 0 1-4.659 0 2.34 2.34 0 0 0-3.32-1.915 2.34 2.34 0 0 1-2.33-4.033 2.34 2.34 0 0 0 0-3.831A2.34 2.34 0 0 1 6.35 6.051a2.34 2.34 0 0 0 3.319-1.915"/><circle cx="12" cy="12" r="3"/></svg>
      </button>
    </div>
    
    <dialog ref="dialog" id="dialog" aria-labelledby="dialog-title" class="fixed inset-0 overflow-y-auto bg-transparent size-auto max-h-none max-w-none">
      
      <div data-closed ref="dialogbackdrop" class="fixed inset-0 transition-opacity duration-150 bg-gray-800/75 data-closed:opacity-0" @click="closeDialog"></div>
      
      <div tabindex="0" class="flex items-center justify-center h-full max-h-screen min-h-full p-4 text-center focus:outline-none sm:p-0">
        
        <div data-closed ref="dialogpanel" class="relative h-auto max-h-screen text-left transition-all duration-150 transform bg-white rounded-lg shadow-xl dark:bg-black dark:text-gray-50 data-closed:translate-y-4 data-closed:opacity-0 sm:my-8 sm:w-full sm:max-w-lg data-closed:sm:translate-y-0 data-closed:sm:scale-95">
          
          <div class="flex flex-col max-h-[calc(100vh-(--spacing(6)))] pt-4 pb-6 sm:pb-8 sm:pt-4">
            
            <button type="button" @click="closeDialog" class="absolute flex items-center justify-center rounded-md cursor-pointer size-10 top-4 right-4">
              <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-x-icon lucide-x size-5"><path d="M18 6 6 18"/><path d="m6 6 12 12"/></svg>
            </button>
            
            <div class="px-4 overflow-y-auto sm:px-6" style="min-height:1px;">
            
              <h3 class="text-2xl font-medium text-gray-600 dark:text-gray-300">
                Settings
              </h3>
              
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
              
              <div class="flex flex-wrap items-start gap-8 mt-6">
                
                <div>
                  <label for="st1" class="block text-sm font-semibold text-gray-700 dark:text-gray-200">Skin Tone 1</label>
                  <select v-model="settings.st1" id="st1" class="mt-1 rounded-md bg-white dark:bg-gray-800 pl-3 py-1.5 text-base text-black dark:text-gray-100 outline-1 -outline-offset-1 outline-white/10 focus:outline-0 focus-visible:-outline-offset-2 focus-visible:outline-indigo-500 sm:text-sm/6">
                    <option :value="null">✋ Base</option>
                    <option value="lst">✋🏻 Light</option>
                    <option value="mlst">✋🏼 Medium-Light</option>
                    <option value="mst">✋🏽 Medium</option>
                    <option value="mdst">✋🏾 Medium-Dark</option>
                    <option value="dst">✋🏿 Dark</option>
                  </select>
                </div>
                
                <div>
                  <label for="st2" class="block text-sm font-semibold text-gray-700 dark:text-gray-200">Skin Tone 2</label>
                  <select v-model="settings.st2" id="st2" class="mt-1 rounded-md bg-white dark:bg-gray-800 pl-3 py-1.5 text-base text-black dark:text-gray-100 outline-1 -outline-offset-1 outline-white/10 focus:outline-0 focus-visible:-outline-offset-2 focus-visible:outline-indigo-500 sm:text-sm/6 disabled:opacity-30" :disabled="!settings.st1">
                    <option :value="null" :disabled="settings.st1">✋ Base</option>
                    <option value="lst">✋🏻 Light</option>
                    <option value="mlst">✋🏼 Medium-Light</option>
                    <option value="mst">✋🏽 Medium</option>
                    <option value="mdst">✋🏾 Medium-Dark</option>
                    <option value="dst">✋🏿 Dark</option>
                  </select>
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
              
              <div class="flex flex-wrap items-center gap-8 mt-8">
                
                <label for="show_unsupported_emojis" class="flex items-center gap-2 cursor-pointer">
                  <input id="show_unsupported_emojis" type="checkbox" v-model="settings.show_unsupported_emojis" class="rounded-md" />
                  Show unsupported emojis (may not be accurate)
                </label>
          
              </div>
              
              <h3 class="mt-8 text-2xl font-medium text-gray-600 dark:text-gray-300">
                Controls
              </h3>
              
              <p class="mt-3 text-sm">
                Click or <span class="font-semibold">Enter</span> to copy an emoji and close the app.
              </p>
              <p class="mt-3 text-sm">
                Hold <span class="font-semibold">Shift</span> to copy multiple emojis. Copied emojis will be displayed at the bottom of the screen. Click any copied emoji from this list to remove it from your currently copied emojis.
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
              
              <h3 class="mt-8 text-2xl font-medium text-gray-600 dark:text-gray-300">
                About
              </h3>
              
              <p class="mt-3 text-sm">
                App version: {{ appVersion }}
              </p>
              
              <p class="mt-3 text-sm">
                Source code: <a href="https://github.com/cliambrown/taurimoji" target="_blank" class="text-blue-600 underline dark:text-blue-300">github.com/cliambrown/taurimoji</a>
              </p>
              
              <div class="mb-12"></div>
              
            </div>
            
          </div>
          
        </div>
        
      </div>
      
    </dialog>
    
  </main>
</template>
