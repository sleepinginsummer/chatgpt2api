<template>
  <ModalShell
    :open="Boolean(file)"
    aria-label="图片预览"
    close-on-overlay
    close-on-escape
    overlay-class="lightbox"
    root-class="lightbox-content"
    size-class=""
    max-width="92vw"
    :z-index="420"
    bare
    @close="emit('close')"
  >
    <template v-if="file">
      <CloseButton class="lightbox-close" label="关闭预览" tone="dark" @click="emit('close')" />
      <div class="lightbox-media-stage">
        <button
          v-if="showNavigation"
          type="button"
          class="lightbox-nav is-previous"
          title="上一张"
          aria-label="上一张"
          @click="emit('previous')"
        >
          <Icon icon="lucide:chevron-left" />
        </button>
        <img
          :src="imageUrl"
          :alt="file.filename"
          class="lightbox-media"
        />
        <button
          v-if="showNavigation"
          type="button"
          class="lightbox-nav is-next"
          title="下一张"
          aria-label="下一张"
          @click="emit('next')"
        >
          <Icon icon="lucide:chevron-right" />
        </button>
        <span v-if="showNavigation && positionLabel" class="lightbox-position">{{ positionLabel }}</span>
      </div>
      <div class="lightbox-info">
          <span class="max-w-[24rem] truncate" :title="file.path">{{ file.filename }}</span>
          <span v-if="sizeLabel">{{ sizeLabel }}</span>
          <span v-if="file.created_at">{{ file.created_at }}</span>
          <button v-if="canShowDownload" class="lightbox-btn" @click="emitFile('download')">
            <Icon icon="lucide:download" />
            下载
          </button>
          <button v-if="canShowCopy" class="lightbox-btn" @click="emitFile('copy')">
            <Icon :icon="copied ? 'lucide:check' : 'lucide:copy'" />
            {{ copied ? '已复制' : '复制链接' }}
          </button>
          <button v-if="canShowTag" class="lightbox-btn" @click="emitFile('edit-tags')">
            <Icon icon="lucide:tag" />
            标签
          </button>
        </div>
    </template>
  </ModalShell>
</template>

<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted } from 'vue'
import { Icon } from '@iconify/vue'
import { CloseButton, ModalShell } from 'nanocat-ui'
import type { GalleryFile } from '@/api/gallery'

const props = withDefaults(defineProps<{
  file: GalleryFile | null
  imageUrl: string
  sizeLabel: string
  copied: boolean
  showActions?: boolean
  showDownloadAction?: boolean
  showCopyAction?: boolean
  showTagAction?: boolean
  showNavigation?: boolean
  positionLabel?: string
}>(), {
  showActions: true,
  showDownloadAction: true,
  showCopyAction: true,
  showTagAction: true,
  showNavigation: false,
  positionLabel: '',
})

const emit = defineEmits<{
  (e: 'close'): void
  (e: 'download', file: GalleryFile): void
  (e: 'copy', file: GalleryFile): void
  (e: 'edit-tags', file: GalleryFile): void
  (e: 'previous'): void
  (e: 'next'): void
}>()

const canShowDownload = computed(() => props.showActions && props.showDownloadAction)
const canShowCopy = computed(() => props.showActions && props.showCopyAction)
const canShowTag = computed(() => props.showActions && props.showTagAction)

function handleNavigationKeydown(event: KeyboardEvent) {
  if (!props.file || !props.showNavigation) return
  if (event.key === 'ArrowLeft') {
    event.preventDefault()
    emit('previous')
  } else if (event.key === 'ArrowRight') {
    event.preventDefault()
    emit('next')
  }
}

onMounted(() => window.addEventListener('keydown', handleNavigationKeydown))
onBeforeUnmount(() => window.removeEventListener('keydown', handleNavigationKeydown))

function emitFile(event: 'download' | 'copy' | 'edit-tags') {
  if (!props.file) return
  emit(event, props.file)
}
</script>

<style scoped>
:global(.lightbox) {
  position: fixed;
  inset: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 24px;
  background: rgba(0, 0, 0, 0.62);
  backdrop-filter: blur(10px);
}

:global(.lightbox-content) {
  position: relative;
  display: flex;
  max-height: 92vh;
  width: fit-content;
  flex-direction: column;
  align-items: center;
  overflow: visible;
  border: 0;
  border-radius: 0;
  background: transparent;
  box-shadow: none;
}

.lightbox-close {
  position: absolute;
  top: -40px;
  right: -4px;
}

.lightbox-media-stage {
  position: relative;
  display: flex;
  width: min(88vw, 80rem);
  max-width: 100%;
  align-items: center;
  justify-content: center;
}

.lightbox-nav {
  position: absolute;
  z-index: 2;
  top: 50%;
  display: inline-flex;
  width: 44px;
  height: 44px;
  align-items: center;
  justify-content: center;
  border: 1px solid rgba(255, 255, 255, 0.45);
  border-radius: 50%;
  background: rgba(12, 18, 28, 0.72);
  color: white;
  cursor: pointer;
  transform: translateY(-50%);
  transition: background 0.15s, border-color 0.15s;
}

.lightbox-nav:hover {
  border-color: rgba(255, 255, 255, 0.8);
  background: rgba(12, 18, 28, 0.9);
}

.lightbox-nav:focus-visible {
  outline: 2px solid white;
  outline-offset: 2px;
}

.lightbox-nav.is-previous {
  left: 12px;
}

.lightbox-nav.is-next {
  right: 12px;
}

.lightbox-nav :deep(svg) {
  width: 24px;
  height: 24px;
}

.lightbox-position {
  position: absolute;
  z-index: 2;
  bottom: 12px;
  left: 50%;
  padding: 4px 9px;
  border-radius: 999px;
  background: rgba(12, 18, 28, 0.72);
  color: white;
  font-size: 12px;
  line-height: 1;
  transform: translateX(-50%);
}

.lightbox-media {
  width: 100%;
  max-width: 100%;
  max-height: 80vh;
  border-radius: var(--gallery-radius, 16px);
  object-fit: contain;
}

@media (max-width: 720px) {
  :global(.lightbox) {
    padding: 16px;
  }

  .lightbox-media-stage {
    width: calc(100vw - 32px);
  }

  .lightbox-media {
    max-height: 76vh;
  }

  .lightbox-nav.is-previous {
    left: 8px;
  }

  .lightbox-nav.is-next {
    right: 8px;
  }
}

.lightbox-info {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  justify-content: center;
  gap: 10px;
  margin-top: 12px;
  font-size: 12px;
  color: rgba(255, 255, 255, 0.78);
}

.lightbox-btn {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  padding: 5px 10px;
  border: 1px solid rgba(255, 255, 255, 0.35);
  border-radius: 999px;
  background: transparent;
  color: white;
  font-size: 11px;
  cursor: pointer;
  transition: background 0.15s, border-color 0.15s;
}

.lightbox-btn:hover {
  border-color: rgba(255, 255, 255, 0.65);
  background: rgba(255, 255, 255, 0.1);
}
</style>
