<script setup>
import { ref, reactive, computed, watch, onMounted, onBeforeUnmount } from 'vue';
import WatermarkTuner from './WatermarkTuner.vue';

let enginePromise = null;
function getEngine() {
  if (!enginePromise) {
    enginePromise = import('../engine/videoEngine.js').then(({ VideoWatermarkEngine }) =>
      VideoWatermarkEngine.create()
    );
  }
  return enginePromise;
}

const fileInput = ref(null);
const dragOver = ref(false);
const supported = ref(true);
const status = ref('idle'); // idle | loading | preview | batch
const isProcessingQueue = ref(false);
const isZipping = ref(false);

// Filename customization
const filenamePrefix = ref('clean_');
const useOriginalName = ref(false);

// Batch items: Array of { id, file, displayName, status, progress, errorMsg, originalUrl, resultUrl, blob, ext, width, height }
const items = ref([]);

// Advanced "tune-it-yourself" mode (off = one-click auto removal)
const advanced = ref(false);

// Watermark position presets for Veo videos. Each carries its own tuned settings.
const VIDEO_PRESETS = [
  {
    id: 'veo',
    label: 'Veo videos (default)',
    desc: 'Current Veo downloads — watermark slightly inset from the bottom-right corner.',
    settings: { gain: 0.6, offsetX: -24, offsetY: -24, sizeScale: 1 },
  },
  {
    id: 'corner',
    label: 'Classic corner',
    desc: 'Watermark right in the bottom-right corner.',
    settings: { gain: 0.6, offsetX: 0, offsetY: 0, sizeScale: 1 },
  },
];
const presetId = ref('veo');
const currentPreset = computed(() => VIDEO_PRESETS.find((p) => p.id === presetId.value));
const settings = reactive({ ...VIDEO_PRESETS[0].settings });

// Switching preset re-seeds the sliders with that preset's settings.
watch(presetId, () => {
  Object.assign(settings, currentPreset.value.settings);
});

// Computed properties for items
const doneItems = computed(() => items.value.filter((i) => i.status === 'done'));
const failedItems = computed(() => items.value.filter((i) => i.status === 'error'));
const hasDoneVideos = computed(() => doneItems.value.length > 0);
const hasFailedVideos = computed(() => failedItems.value.length > 0);
const isProcessingAny = computed(() => isProcessingQueue.value || items.value.some((i) => i.status === 'processing' || i.status === 'pending'));

// Dynamic output filename calculation
function getOutputName(file, ext = 'mp4') {
  if (!file) return `clean_video.${ext}`;
  const rawBase = file.name.replace(/\.[^/.]+$/, '');
  const prefix = useOriginalName.value ? '' : (filenamePrefix.value || '');
  return `${prefix}${rawBase}.${ext}`;
}

// Preview state for Advanced mode
const frame = ref(null); // { width, height, imageData }
const base = ref(null);
const bgImg = ref(null);
let engine = null;

onMounted(async () => {
  const { VideoWatermarkEngine } = await import('../engine/videoEngine.js');
  supported.value = VideoWatermarkEngine.isSupported();
});

onBeforeUnmount(() => {
  reset();
});

function openPicker() {
  fileInput.value?.click();
}

function onDrop(e) {
  dragOver.value = false;
  handleFiles(e.dataTransfer.files);
}

function onChange(e) {
  handleFiles(e.target.files);
}

async function handleFiles(fileList) {
  const valid = Array.from(fileList).filter((f) =>
    f.type.startsWith('video/') || /\.(mp4|webm|mov|mkv|m4v|avi|ogv)$/i.test(f.name)
  );
  if (!valid.length) {
    if (fileList.length > 0) {
      alert('Please upload valid video files (MP4, WebM, MOV, MKV).');
    }
    return;
  }

  // Clear previous session URLs if transitioning from idle or finished state
  if (status.value === 'idle') {
    resetUrls();
    items.value = [];
  }

  // Add all files into the items queue
  const newItems = valid.map((file) => ({
    id: Math.random().toString(36).slice(2, 9),
    file,
    displayName: file.name,
    status: 'pending', // pending | processing | done | error
    progress: 0,
    errorMsg: '',
    originalUrl: '',
    resultUrl: '',
    blob: null,
    ext: 'mp4',
    width: 0,
    height: 0,
  }));
  items.value.push(...newItems);

  try {
    engine = await getEngine();
    if (advanced.value && !frame.value) {
      status.value = 'loading';
      const f = await grabPreviewFrame(valid[0]);
      frame.value = f;
      base.value = engine.getVeoWatermark(f.width, f.height);
      bgImg.value = engine.sparkleImage;
      status.value = 'preview';
      return;
    }

    status.value = 'batch';
    processQueue();
  } catch (e) {
    console.error(e);
    alert(e?.message || 'Could not load video watermark engine.');
    status.value = items.value.length ? 'batch' : 'idle';
  }
}

// Decode a representative frame with a <video> element with a safety timeout
function grabPreviewFrame(file) {
  return new Promise((resolve, reject) => {
    const url = URL.createObjectURL(file);
    const v = document.createElement('video');
    v.preload = 'auto';
    v.muted = true;
    v.playsInline = true;
    v.src = url;

    let timer = null;
    const cleanup = () => {
      if (timer) clearTimeout(timer);
      URL.revokeObjectURL(url);
    };

    timer = setTimeout(() => {
      cleanup();
      reject(new Error('Timed out reading preview frame from video.'));
    }, 10000);

    v.onerror = () => {
      cleanup();
      reject(new Error('Could not read this video file.'));
    };

    v.onloadedmetadata = () => {
      const dur = Number.isFinite(v.duration) && v.duration > 0 ? v.duration : 1;
      const seekTo = Math.min(Math.max(dur * 0.3, 0.1), Math.max(dur - 0.05, 0.1));
      const onSeeked = () => {
        try {
          const w = v.videoWidth || 1280;
          const h = v.videoHeight || 720;
          const c = document.createElement('canvas');
          c.width = w;
          c.height = h;
          const cx = c.getContext('2d', { willReadFrequently: true });
          cx.drawImage(v, 0, 0, w, h);
          const imageData = cx.getImageData(0, 0, w, h);
          cleanup();
          resolve({ width: w, height: h, imageData });
        } catch (err) {
          cleanup();
          reject(err);
        }
      };
      v.onseeked = onSeeked;
      try {
        v.currentTime = seekTo;
      } catch {
        onSeeked();
      }
    };
  });
}

function resetSettings() {
  Object.assign(settings, currentPreset.value.settings);
}

// Sequential queue processor
async function processQueue() {
  if (isProcessingQueue.value) return;
  isProcessingQueue.value = true;

  try {
    if (!engine) engine = await getEngine();

    for (const item of items.value) {
      if (item.status !== 'pending') continue;

      item.status = 'processing';
      item.progress = 0;
      item.errorMsg = '';

      try {
        const result = await engine.process(item.file, {
          ...settings,
          onProgress: ({ progress: p }) => {
            item.progress = p;
          },
        });

        item.originalUrl = result.originalUrl;
        item.resultUrl = result.url;
        item.blob = result.blob;
        item.ext = result.ext || 'mp4';
        item.width = result.width || 0;
        item.height = result.height || 0;
        item.status = 'done';
      } catch (err) {
        console.error(`Error processing ${item.displayName}:`, err);
        item.status = 'error';
        item.errorMsg = err?.message || 'Processing failed';
      }
    }
  } finally {
    isProcessingQueue.value = false;
  }
}

// Confirm and start export from Advanced Tuner
function startBatchExport() {
  status.value = 'batch';
  processQueue();
}

// Return to preview tuner for re-adjustments
function backToPreview() {
  status.value = 'preview';
}

// Retry a single failed item
function retryItem(item) {
  if (isProcessingQueue.value) return;
  item.status = 'pending';
  item.errorMsg = '';
  item.progress = 0;
  processQueue();
}

// Retry all failed items
function retryFailed() {
  if (isProcessingQueue.value) return;
  items.value.forEach((item) => {
    if (item.status === 'error') {
      item.status = 'pending';
      item.errorMsg = '';
      item.progress = 0;
    }
  });
  processQueue();
}

// Download single item (Firefox-safe)
function downloadItem(item) {
  if (!item.resultUrl) return;
  const a = document.createElement('a');
  a.href = item.resultUrl;
  a.download = getOutputName(item.file, item.ext);
  document.body.appendChild(a);
  a.click();
  document.body.removeChild(a);
}

// Download all completed items (ZIP if multiple, direct if single)
async function downloadAll() {
  const done = items.value.filter((i) => i.status === 'done' && i.blob);
  if (!done.length) return;

  if (done.length === 1) {
    downloadItem(done[0]);
    return;
  }

  isZipping.value = true;
  try {
    const { default: JSZip } = await import('https://cdn.jsdelivr.net/npm/jszip@3.10.1/+esm');
    const zip = new JSZip();
    done.forEach((item) => {
      zip.file(getOutputName(item.file, item.ext), item.blob);
    });

    const zipBlob = await zip.generateAsync({ type: 'blob' });
    const url = URL.createObjectURL(zipBlob);
    const a = document.createElement('a');
    a.href = url;
    a.download = `cleaned_videos_${Date.now()}.zip`;
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
    setTimeout(() => URL.revokeObjectURL(url), 2000);
  } catch (err) {
    console.error('Failed to create ZIP:', err);
    alert('Failed to generate ZIP archive. You can download videos individually.');
  } finally {
    isZipping.value = false;
  }
}

// Remove an item from the list
function removeItem(index) {
  const item = items.value[index];
  if (item) {
    if (item.resultUrl) URL.revokeObjectURL(item.resultUrl);
    if (item.originalUrl) URL.revokeObjectURL(item.originalUrl);
    items.value.splice(index, 1);
  }
  if (!items.value.length) {
    reset();
  }
}

function resetUrls() {
  items.value.forEach((item) => {
    if (item.resultUrl) URL.revokeObjectURL(item.resultUrl);
    if (item.originalUrl) URL.revokeObjectURL(item.originalUrl);
  });
}

function reset() {
  resetUrls();
  items.value = [];
  frame.value = null;
  status.value = 'idle';
  if (fileInput.value) fileInput.value.value = '';
}
</script>

<template>
  <div
    class="max-w-5xl mx-auto bg-white dark:bg-theme-cardDark rounded-3xl shadow-xl dark:shadow-none p-4 border border-gray-100 dark:border-gray-800 relative z-10 transition-colors"
  >
    <!-- Unsupported -->
    <div
      v-if="!supported"
      class="flex flex-col items-center justify-center w-full h-56 rounded-2xl bg-red-50/60 dark:bg-red-900/10 border border-red-200 dark:border-red-900/30 text-center px-6"
    >
      <iconify-icon icon="ph:warning-circle-bold" width="36" class="text-red-500 mb-2"></iconify-icon>
      <p class="font-bold text-red-600 dark:text-red-400">Your browser can't process video locally.</p>
      <p class="text-sm text-red-500/80 mt-1">Please try the latest Chrome or Edge on desktop.</p>
    </div>

    <!-- Upload (Idle) -->
    <div
      v-else-if="status === 'idle'"
      class="group relative flex flex-col items-center justify-center w-full min-h-[14rem] py-8 border-2 border-dashed rounded-2xl bg-gray-50/50 dark:bg-gray-800/50 transition-all cursor-pointer"
      :class="dragOver ? 'border-brand-primary bg-indigo-50/60 dark:bg-gray-800' : 'border-gray-300 dark:border-gray-700 hover:bg-indigo-50/50 dark:hover:bg-gray-800 hover:border-brand-primary'"
      role="button" tabindex="0" aria-label="Upload videos"
      @click="openPicker" @keydown.enter="openPicker"
      @dragover.prevent="dragOver = true" @dragenter.prevent="dragOver = true"
      @dragleave.prevent="dragOver = false" @drop.prevent="onDrop"
    >
      <div class="flex flex-col items-center justify-center text-center px-4">
        <div class="w-14 h-14 bg-white dark:bg-gray-700 rounded-full shadow-sm flex items-center justify-center mb-3 group-hover:scale-110 transition-transform">
          <iconify-icon icon="ph:video-camera-bold" class="text-2xl text-gray-400 dark:text-gray-300 group-hover:text-brand-primary" aria-hidden="true"></iconify-icon>
        </div>
        <p class="mb-1 text-base font-bold text-slate-700 dark:text-slate-200 group-hover:text-brand-primary transition-colors">
          Click to upload or drag Gemini Veo videos
        </p>
        <p class="text-sm text-slate-400 dark:text-slate-500">MP4, WebM, MOV · Multiple files supported · Audio preserved</p>

        <div class="mt-4 flex flex-col items-center gap-1.5" @click.stop>
          <label class="inline-flex items-center gap-2 text-xs font-semibold text-slate-500 dark:text-slate-400">
            Watermark position:
            <select
              v-model="presetId"
              class="text-xs font-semibold bg-white dark:bg-gray-800 border border-gray-200 dark:border-gray-700 rounded-lg px-2 py-1 text-slate-700 dark:text-slate-200 focus:outline-none focus:ring-2 focus:ring-brand-primary/50 cursor-pointer"
            >
              <option v-for="p in VIDEO_PRESETS" :key="p.id" :value="p.id">{{ p.label }}</option>
            </select>
          </label>
          <p class="text-[11px] text-slate-400 dark:text-slate-500 max-w-xs">{{ currentPreset.desc }}</p>
        </div>

        <label class="mt-3 inline-flex items-center gap-2 text-xs font-semibold text-slate-500 dark:text-slate-400 cursor-pointer" @click.stop>
          <input type="checkbox" v-model="advanced" class="accent-brand-primary w-3.5 h-3.5" />
          Advanced: tune it yourself
        </label>
      </div>
      <input ref="fileInput" type="file" accept="video/*" multiple class="hidden" aria-label="Video file input" @change="onChange" />
    </div>

    <!-- Loading the preview frame -->
    <div v-else-if="status === 'loading'" class="flex flex-col items-center justify-center w-full h-56">
      <div class="w-12 h-12 rounded-full border-4 border-transparent border-t-brand-primary border-r-brand-secondary border-b-brand-accent animate-spin mb-3"></div>
      <p class="font-bold text-brand-primary">Loading preview…</p>
    </div>

    <!-- Preview + manual controls for Advanced mode -->
    <div v-else-if="status === 'preview'" class="animate-fade-in">
      <div class="flex flex-col lg:flex-row gap-6">
        <div class="flex-1 min-w-0">
          <WatermarkTuner :settings="settings" :frame="frame" :bg-img="bgImg" :base="base" />
          <p class="text-xs text-slate-400 dark:text-slate-500 mt-3 leading-relaxed">
            Adjust the sliders until the watermark disappears in the zoomed corner. The
            <span class="text-brand-primary font-semibold">blue box</span> shows what gets cleaned.
          </p>
        </div>

        <div class="w-full lg:w-60 flex-shrink-0">
          <div class="bg-white dark:bg-theme-cardDark rounded-2xl shadow-lg border border-gray-100 dark:border-gray-800 p-5 space-y-3 sticky top-24">
            <h2 class="font-bold text-slate-900 dark:text-white text-base">Export</h2>
            <label class="block">
              <div class="text-xs font-bold text-slate-600 dark:text-slate-300 mb-1">Position preset</div>
              <select
                v-model="presetId"
                class="w-full text-xs font-semibold bg-white dark:bg-gray-800 border border-gray-200 dark:border-gray-700 rounded-lg px-2 py-1.5 text-slate-700 dark:text-slate-200 focus:outline-none focus:ring-2 focus:ring-brand-primary/50 cursor-pointer"
              >
                <option v-for="p in VIDEO_PRESETS" :key="p.id" :value="p.id">{{ p.label }}</option>
              </select>
              <p class="text-[11px] text-slate-400 dark:text-slate-500 mt-1">{{ currentPreset.desc }}</p>
            </label>

            <button @click="resetSettings" class="w-full text-xs font-semibold text-slate-500 hover:text-brand-primary transition-colors">
              Reset sliders to preset
            </button>

            <button @click="startBatchExport" class="group w-full py-3 relative overflow-hidden rounded-xl font-bold text-white shadow-lg shadow-brand-primary/30 transition-all">
              <div class="absolute inset-0 bg-gradient-to-r from-brand-primary via-brand-secondary to-brand-accent group-hover:scale-110 transition-transform duration-500"></div>
              <div class="relative flex items-center justify-center gap-2">
                <iconify-icon icon="ph:sparkle-fill" width="18"></iconify-icon> Remove &amp; Export
              </div>
            </button>

            <button @click="reset" class="w-full py-2.5 bg-white dark:bg-gray-800 border border-gray-200 dark:border-gray-700 text-slate-600 dark:text-slate-300 hover:border-brand-primary hover:text-brand-primary rounded-xl font-bold transition-all">
              Choose another video
            </button>
          </div>
        </div>
      </div>
    </div>

    <!-- Batch List View -->
    <div v-else class="text-left animate-fade-in">
      <!-- Top toolbar container using space-between -->
      <div
        class="flex flex-wrap items-center justify-between gap-3 p-4 mb-5 bg-gray-50/90 dark:bg-gray-800/60 rounded-2xl border border-gray-200/80 dark:border-gray-700/80"
      >
        <!-- Left: Filename prefix and Original Name checkbox -->
        <div class="flex flex-wrap items-center gap-3 sm:gap-4">
          <div class="flex items-center gap-2">
            <span class="text-xs font-bold text-slate-600 dark:text-slate-300 whitespace-nowrap">Prefix:</span>
            <input
              v-model="filenamePrefix"
              :disabled="useOriginalName"
              type="text"
              placeholder="clean_"
              class="w-24 sm:w-28 text-xs font-mono font-medium px-2.5 py-1.5 rounded-lg border border-gray-200 dark:border-gray-700 bg-white dark:bg-gray-800 text-slate-800 dark:text-slate-100 disabled:opacity-40 disabled:cursor-not-allowed focus:outline-none focus:ring-2 focus:ring-brand-primary/50 transition-all"
            />
          </div>

          <label
            class="inline-flex items-center gap-1.5 text-xs font-semibold text-slate-600 dark:text-slate-300 cursor-pointer select-none group"
            title="use original name"
          >
            <input
              type="checkbox"
              v-model="useOriginalName"
              class="accent-brand-primary w-4 h-4 rounded cursor-pointer"
              title="use original name"
            />
            <span class="group-hover:text-brand-primary transition-colors">Use original name</span>
          </label>
        </div>

        <!-- Right: Retry Failed & Download All buttons -->
        <div class="flex flex-wrap items-center gap-2">
          <!-- Retry Failed button -->
          <button
            v-if="hasFailedVideos"
            @click="retryFailed"
            :disabled="isProcessingQueue"
            class="inline-flex items-center gap-1.5 px-3 py-1.5 bg-amber-500 hover:bg-amber-600 disabled:opacity-50 text-white text-xs font-bold rounded-xl transition-all shadow-sm active:scale-95 cursor-pointer"
            title="Retry failed videos"
          >
            <iconify-icon icon="ph:arrow-clockwise-bold" width="15"></iconify-icon>
            Retry Failed ({{ failedItems.length }})
          </button>

          <!-- Download All button -->
          <button
            v-if="hasDoneVideos"
            @click="downloadAll"
            :disabled="isZipping"
            class="inline-flex items-center gap-1.5 px-4 py-1.5 bg-green-600 hover:bg-green-700 disabled:opacity-50 text-white text-xs font-bold rounded-xl transition-all shadow-sm active:scale-95 shadow-green-600/20 cursor-pointer"
            title="Download all successful videos"
          >
            <iconify-icon v-if="isZipping" icon="ph:spinner-gap-bold" class="animate-spin" width="15"></iconify-icon>
            <iconify-icon v-else icon="ph:download-simple-bold" width="15"></iconify-icon>
            {{ isZipping ? 'Zipping…' : `Download All (${doneItems.length})` }}
          </button>

          <!-- Add more videos button -->
          <button
            @click="openPicker"
            class="inline-flex items-center gap-1 px-3 py-1.5 bg-white dark:bg-gray-800 border border-gray-200 dark:border-gray-700 hover:border-brand-primary hover:text-brand-primary text-slate-600 dark:text-slate-300 text-xs font-bold rounded-xl transition-all cursor-pointer"
          >
            <iconify-icon icon="ph:plus-bold" width="14"></iconify-icon>
            Add
          </button>

          <!-- Adjust tuner button (if advanced) -->
          <button
            v-if="advanced && frame"
            @click="backToPreview"
            class="inline-flex items-center gap-1 px-3 py-1.5 bg-white dark:bg-gray-800 border border-gray-200 dark:border-gray-700 hover:border-brand-primary hover:text-brand-primary text-slate-600 dark:text-slate-300 text-xs font-bold rounded-xl transition-all cursor-pointer"
          >
            <iconify-icon icon="ph:sliders-bold" width="14"></iconify-icon>
            Tune
          </button>

          <!-- Clear / Reset button -->
          <button
            @click="reset"
            class="inline-flex items-center gap-1 px-3 py-1.5 bg-white dark:bg-gray-800 border border-gray-200 dark:border-gray-700 text-slate-500 hover:text-red-500 hover:border-red-300 dark:hover:border-red-800 text-xs font-bold rounded-xl transition-all cursor-pointer"
            title="Clear all videos and restart"
          >
            Clear
          </button>
        </div>
      </div>

      <!-- Video Rows List -->
      <div class="space-y-4">
        <div
          v-for="(item, idx) in items"
          :key="item.id"
          class="p-4 bg-white dark:bg-theme-cardDark rounded-2xl shadow-sm border border-gray-100 dark:border-gray-800 flex flex-col gap-3 transition-all"
        >
          <!-- Item Row Header: space-between -->
          <div class="flex flex-wrap items-center justify-between gap-2">
            <!-- Left info: file icon + names -->
            <div class="flex items-center gap-2.5 min-w-0 max-w-full sm:max-w-xl">
              <div
                class="w-8 h-8 rounded-lg flex items-center justify-center flex-shrink-0 text-base"
                :class="{
                  'bg-gray-100 dark:bg-gray-800 text-gray-500': item.status === 'pending',
                  'bg-indigo-50 dark:bg-indigo-900/30 text-brand-primary': item.status === 'processing',
                  'bg-green-50 dark:bg-green-900/30 text-green-600': item.status === 'done',
                  'bg-red-50 dark:bg-red-900/30 text-red-500': item.status === 'error',
                }"
              >
                <iconify-icon
                  :icon="
                    item.status === 'done'
                      ? 'ph:check-circle-fill'
                      : item.status === 'error'
                      ? 'ph:warning-circle-fill'
                      : 'ph:video-camera-bold'
                  "
                ></iconify-icon>
              </div>

              <div class="min-w-0 truncate">
                <div class="flex items-center gap-2">
                  <span class="font-bold text-sm text-slate-800 dark:text-slate-100 truncate" :title="item.displayName">
                    {{ item.displayName }}
                  </span>
                  <span class="text-[10px] font-mono text-slate-400 dark:text-slate-500 flex-shrink-0">
                    ({{ (item.file.size / (1024 * 1024)).toFixed(1) }} MB)
                  </span>
                </div>
                <div class="text-[11px] font-mono text-slate-400 dark:text-slate-500 truncate">
                  Output: {{ getOutputName(item.file, item.ext) }}
                </div>
              </div>
            </div>

            <!-- Right badges & actions -->
            <div class="flex items-center gap-2">
              <!-- Pending badge -->
              <span
                v-if="item.status === 'pending'"
                class="inline-flex items-center gap-1 text-xs font-semibold px-2.5 py-1 rounded-full bg-gray-100 dark:bg-gray-800 text-gray-500 dark:text-gray-400"
              >
                <iconify-icon icon="ph:hourglass-bold" width="13"></iconify-icon>
                Waiting in queue
              </span>

              <!-- Processing badge -->
              <span
                v-else-if="item.status === 'processing'"
                class="inline-flex items-center gap-1.5 text-xs font-semibold px-2.5 py-1 rounded-full bg-indigo-50 dark:bg-indigo-900/30 text-brand-primary"
              >
                <div class="w-3.5 h-3.5 rounded-full border-2 border-transparent border-t-brand-primary border-r-brand-secondary animate-spin"></div>
                Cleaning… {{ Math.round(item.progress * 100) }}%
              </span>

              <!-- Error state & single retry -->
              <div v-else-if="item.status === 'error'" class="flex items-center gap-2">
                <span class="inline-flex items-center gap-1 text-xs font-bold px-2.5 py-1 rounded-full bg-red-50 dark:bg-red-900/30 text-red-600 dark:text-red-400">
                  <iconify-icon icon="ph:warning-circle-bold" width="13"></iconify-icon>
                  Failed
                </span>
                <button
                  @click="retryItem(item)"
                  class="inline-flex items-center gap-1 px-2.5 py-1 bg-amber-500 hover:bg-amber-600 text-white text-xs font-bold rounded-lg transition-all active:scale-95 cursor-pointer"
                >
                  <iconify-icon icon="ph:arrow-clockwise-bold" width="12"></iconify-icon>
                  Retry
                </button>
              </div>

              <!-- Done state: download button -->
              <div v-else-if="item.status === 'done'" class="flex items-center gap-2">
                <span class="inline-flex items-center gap-1 text-xs font-bold px-2.5 py-1 rounded-full bg-green-50 dark:bg-green-900/30 text-green-600 dark:text-green-400">
                  <iconify-icon icon="ph:check-circle-fill" width="13"></iconify-icon>
                  Cleaned
                </span>
                <button
                  @click="downloadItem(item)"
                  class="inline-flex items-center gap-1.5 px-3 py-1 bg-green-600 hover:bg-green-700 text-white text-xs font-bold rounded-lg transition-all active:scale-95 cursor-pointer shadow-sm"
                >
                  <iconify-icon icon="ph:download-simple-bold" width="14"></iconify-icon>
                  Download
                </button>
              </div>

              <!-- Delete / Remove row item -->
              <button
                @click="removeItem(idx)"
                class="p-1 text-slate-400 hover:text-red-500 dark:hover:text-red-400 rounded-lg transition-colors cursor-pointer"
                title="Remove video"
              >
                <iconify-icon icon="ph:trash-bold" width="16"></iconify-icon>
              </button>
            </div>
          </div>

          <!-- Progress bar for processing item -->
          <div v-if="item.status === 'processing'" class="w-full mt-1">
            <div class="w-full h-2 rounded-full bg-gray-100 dark:bg-gray-700 overflow-hidden">
              <div
                class="h-full bg-gradient-to-r from-brand-primary via-brand-secondary to-brand-accent transition-all duration-150"
                :style="{ width: `${Math.round(item.progress * 100)}%` }"
              ></div>
            </div>
          </div>

          <!-- Error message alert -->
          <div
            v-if="item.status === 'error'"
            class="p-3 bg-red-50/70 dark:bg-red-900/20 border border-red-200/70 dark:border-red-900/40 rounded-xl text-xs font-medium text-red-600 dark:text-red-300 flex items-center gap-2"
          >
            <iconify-icon icon="ph:warning-bold" class="text-base flex-shrink-0 text-red-500"></iconify-icon>
            <span>{{ item.errorMsg || 'Something went wrong while processing this video.' }}</span>
          </div>

          <!-- Side-by-side Video Player when Done -->
          <div v-if="item.status === 'done'" class="grid grid-cols-1 md:grid-cols-2 gap-3 min-w-0 mt-1">
            <!-- Original video player -->
            <div class="rounded-xl overflow-hidden border border-gray-200 dark:border-gray-700 bg-white dark:bg-theme-cardDark shadow-sm">
              <div class="bg-gray-50 dark:bg-gray-800/80 px-3 py-1.5 border-b border-gray-200 dark:border-gray-700 font-bold text-xs text-slate-700 dark:text-slate-200 flex justify-between items-center">
                <span>Original</span>
                <span v-if="item.width && item.height" class="text-[10px] font-mono text-slate-400">
                  {{ item.width }} × {{ item.height }} px
                </span>
              </div>
              <div class="p-2 checker flex justify-center bg-black/5">
                <video :src="item.originalUrl" controls playsinline class="max-h-56 w-full object-contain rounded"></video>
              </div>
            </div>

            <!-- Cleaned video player -->
            <div class="rounded-xl overflow-hidden border border-green-500/40 ring-1 ring-green-500/20 bg-white dark:bg-theme-cardDark shadow-sm">
              <div class="bg-green-50 dark:bg-green-900/20 px-3 py-1.5 border-b border-green-500/30 flex items-center justify-between text-xs font-bold text-green-600 dark:text-green-400">
                <span class="flex items-center gap-1">
                  <iconify-icon icon="ph:check-circle-fill" width="14"></iconify-icon> Cleaned
                </span>
                <span v-if="item.blob" class="text-[10px] font-mono text-green-700/70 dark:text-green-300/70">
                  {{ (item.blob.size / (1024 * 1024)).toFixed(1) }} MB
                </span>
              </div>
              <div class="p-2 checker flex justify-center bg-black/5">
                <video :src="item.resultUrl" controls playsinline class="max-h-56 w-full object-contain rounded"></video>
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- Hidden file input for adding more videos -->
      <input ref="fileInput" type="file" accept="video/*" multiple class="hidden" aria-label="Video file input" @change="onChange" />
    </div>
  </div>
</template>
