<template>
    <div ref="editorContainer" class="editor-container"></div>
  </template>
  
  <script setup>
  import { ref, watch, onMounted, onBeforeUnmount } from 'vue'
  import EditorJS from '@editorjs/editorjs'
  import Header from '@editorjs/header'
  import List from '@editorjs/list'
  import Link from '@editorjs/link'
  import Paragraph from '@editorjs/paragraph'
  
  const props = defineProps({
    modelValue: {
      type: Object,
      default: () => ({ blocks: [] })
    }
  })
  
  const emit = defineEmits(['update:modelValue'])
  
  const editorContainer = ref(null)
  let editor = null
  
  const config = {
    tools: {
      header: {
        class: Header,
        config: {
          levels: [1, 2, 3, 4, 5, 6],
          defaultLevel: 3
        }
      },
      list: {
        class: List,
        inlineToolbar: true
      },
      paragraph: {
        class: Paragraph,
        inlineToolbar: true
      },
      link: {
        class: Link,
        config: {
          endpoint: 'http://localhost:8088/fetchUrl'
        }
      }
    },
    data: props.modelValue,
    onChange: async (api) => {
      const output = await api.saver.save()
      emit('update:modelValue', output)
    }
  }
  
  onMounted(() => {
    editor = new EditorJS({
      ...config,
      holder: editorContainer.value
    })
  })
  
  onBeforeUnmount(() => {
    if (editor) {
      editor.destroy()
    }
  })
  
  watch(() => props.modelValue, (newValue) => {
    if (editor) {
      editor.render(newValue)
    }
  }, { deep: true })
  </script>
  
  <style>
  .editor-container {
    min-height: 500px;
    border: 1px solid #ddd;
    border-radius: 8px;
    padding: 1rem;
  }
  
  .codex-editor__redactor {
    padding-bottom: 20px !important;
  }
  </style>