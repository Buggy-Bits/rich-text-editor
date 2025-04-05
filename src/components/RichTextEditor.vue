<script setup>
import { ref, reactive, onMounted, watch, nextTick } from 'vue'
import { Editor, EditorContent } from '@tiptap/vue-3'
import StarterKit from '@tiptap/starter-kit'
import Underline from '@tiptap/extension-underline'
import Strike from '@tiptap/extension-strike'
import Heading from '@tiptap/extension-heading'
import TextAlign from '@tiptap/extension-text-align'
import localforage from 'localforage'
import Placeholder from '@tiptap/extension-placeholder'
import CharacterCount from '@tiptap/extension-character-count'
import Link from '@tiptap/extension-link'
import Color from '@tiptap/extension-color'
import TextStyle from '@tiptap/extension-text-style'

const tabs = reactive([])
const activeTab = ref(null)
const activeEditor = ref(null)


const tabsStorageKey = 'editor-tabs'

onMounted(async () => {
  try {
    const savedTabs = await localforage.getItem(tabsStorageKey)
    if (savedTabs && savedTabs.length) {
      tabs.push(...savedTabs)
      activeTab.value = savedTabs[0].id
    } else {
      addTab()
    }
  } catch (error) {
    console.error('Error loading tabs from storage:', error)
    addTab()
  }
})

const addTab = () => {
  saveCurrentTabContent()

  const newTab = {
    id: Date.now(),
    title: `Document ${tabs.length + 1}`,
    content: '',
    history: {
      undoStack: [],
      redoStack: []
    }
  }
  tabs.push(newTab)
  activeTab.value = newTab.id
  saveTabs()
}

const closeTab = (tabId) => {

  saveCurrentTabContent()

  const index = tabs.findIndex(tab => tab.id === tabId)
  if (index === -1) return

  tabs.splice(index, 1)
  if (activeTab.value === tabId) {
    activeTab.value = tabs.length ? tabs[Math.max(0, index - 1)].id : null
  }
  saveTabs()
}

const saveCurrentTabContent = () => {
  if (activeEditor.value && activeTab.value) {
    const currentTab = tabs.find(tab => tab.id === activeTab.value)
    if (currentTab) {
      currentTab.content = activeEditor.value.getHTML()
    }
  }
}

const switchTab = async (tabId) => {
  if (activeTab.value === tabId) return

  // Save before switching tab
  saveCurrentTabContent()
  if (activeEditor.value) {
    activeEditor.value.destroy()
    activeEditor.value = null
  }
  activeTab.value = tabId
  await nextTick()
  initializeEditor()

  // Save to storage
  saveTabs()
}

const initializeEditor = () => {
  if (!activeTab.value) return

  const currentTab = tabs.find(tab => tab.id === activeTab.value)
  if (!currentTab) return

  // undo /redo
  if (!currentTab.history) {
    currentTab.history = {
      undoStack: [],
      redoStack: []
    }
  }

  activeEditor.value = new Editor({
    content: currentTab.content,
    extensions: [
      StarterKit.configure({
        history: false, 
      }),
      Underline,
      Strike,
      Heading.configure({ levels: [1, 2, 3] }),
      TextAlign.configure({ types: ['paragraph', 'heading'] }),
      Placeholder.configure({ placeholder: 'Start typing...' }),
      CharacterCount,
      Link.configure({
        openOnClick: true, // open on click
        linkOnPaste: true, // auto link on paste
      }),
      Color,
      TextStyle,
    ],
    onUpdate: ({ editor }) => {
      const html = editor.getHTML()
      if (currentTab.content !== html) {
        currentTab.history.undoStack.push(currentTab.content)
        currentTab.history.redoStack = [] // Clear redo stack on new changes
        currentTab.content = html
        saveTabs()
      }
    },
  })
}

const saveTabs = async () => {
  try {
    const tabsToSave = tabs.map(tab => ({
      id: tab.id,
      title: tab.title,
      content: tab.content,
      history: tab.history
    }))

    await localforage.setItem(tabsStorageKey, tabsToSave)
  } catch (error) {
    console.error('Error saving tabs:', error)
  }
}


const handleUndo = () => {
  if (!activeEditor.value) return

  const currentTab = tabs.find(tab => tab.id === activeTab.value)
  if (!currentTab || !currentTab.history || currentTab.history.undoStack.length === 0) return

  const previousContent = currentTab.history.undoStack.pop()

  currentTab.history.redoStack.push(currentTab.content)

  currentTab.content = previousContent
  activeEditor.value.commands.setContent(previousContent, false)

  saveTabs()
}

// custom redo function 
const handleRedo = () => {
  if (!activeEditor.value) return

  const currentTab = tabs.find(tab => tab.id === activeTab.value)
  if (!currentTab || !currentTab.history || currentTab.history.redoStack.length === 0) return

  const nextContent = currentTab.history.redoStack.pop()

  currentTab.history.undoStack.push(currentTab.content)
  currentTab.content = nextContent
  activeEditor.value.commands.setContent(nextContent, false)

  saveTabs()
}

const setFontSize = (size) => {
  if (!activeEditor.value) return
  activeEditor.value.chain().focus().setFontSize(size).run()
}

const changeTextColor = (color) => {
  if (!activeEditor.value) return
  activeEditor.value.chain().focus().setColor(color).run()
}

const insertLink = () => {
  if (!activeEditor.value) return

  const url = prompt('Enter URL:')
  if (url) {
    // Check if URL has protocol, add http:// if missing
    const formattedUrl = url.match(/^https?:\/\//) ? url : `http://${url}`
    activeEditor.value.chain().focus().setLink({ href: formattedUrl, target: '_blank' }).run()
  }
}

const removeLink = () => {
  if (!activeEditor.value) return
  activeEditor.value.chain().focus().unsetLink().run()
}

const applyTextStyle = (style) => {
  if (!activeEditor.value) return

  switch (style) {
    case 'title':
      activeEditor.value.chain().focus().setFontSize('28px').setFontWeight('bold').run()
      break
    case 'heading1':
      activeEditor.value.chain().focus().toggleHeading({ level: 1 }).run()
      break
    case 'heading2':
      activeEditor.value.chain().focus().toggleHeading({ level: 2 }).run()
      break
    case 'heading3':
      activeEditor.value.chain().focus().toggleHeading({ level: 3 }).run()
      break
    case 'normal':
      activeEditor.value.chain().focus().setParagraph().setFontSize('16px').setFontWeight('normal').run()
      break
  }
}

watch(activeTab, (newTabId, oldTabId) => {
  if (newTabId && newTabId !== oldTabId) {
    if (activeEditor.value) {
      // Clean up previous editor instance
      activeEditor.value.destroy()
      activeEditor.value = null
    }

    nextTick(() => {
      initializeEditor()
    })
  }
}, { immediate: true })

window.addEventListener('beforeunload', () => {
  saveCurrentTabContent()
  saveTabs()
})
</script>

<template>
  <div class="editor-container">
    <div class="tab-bar">
      <div v-for="tab in tabs" :key="tab.id" :class="['tab', { 'active': activeTab === tab.id }]"
        @click="switchTab(tab.id)">
        <input v-model="tab.title" @click.stop @change="saveTabs" />
        <button @click.stop="closeTab(tab.id)" class="close-tab" :disabled="tabs.length === 1">
          &times;
        </button>
      </div>
      <div class="new-tab" @click="addTab">+</div>
    </div>
    <div v-if="activeEditor" class="toolbar">
      <!-- Undo/Redo -->
      <div class="toolbar-group">
        <button class="toolbar-button" @click="handleUndo" title="Undo">
          ↩
        </button>
        <button class="toolbar-button" @click="handleRedo" title="Redo">
          ↪
        </button>
      </div>

      <!-- Text Formatting -->
      <div class="toolbar-group">
        <button :class="{ 'active': activeEditor.isActive('bold') }"
          @click="activeEditor.chain().focus().toggleBold().run()"><strong>B</strong></button>
        <button @click="activeEditor.chain().focus().toggleCodeBlock().run()"
          :class="{ 'active': activeEditor.isActive('codeBlock') }">&lt;/&gt;</button>
        <button :class="{ 'active': activeEditor.isActive('italic') }"
          @click="activeEditor.chain().focus().toggleItalic().run()"><em>i</em></button>
        <button :class="{ 'active': activeEditor.isActive('underline') }"
          @click="activeEditor.chain().focus().toggleUnderline().run()"><u>U</u></button>
        <button :class="{ 'active': activeEditor.isActive('strike') }"
          @click="activeEditor.chain().focus().toggleStrike().run()"><del>abc</del></button>
      </div>

      <!-- Text Style Presets -->
      <div class="toolbar-group">
        <select @change="applyTextStyle($event.target.value)">
          <option value="normal">Normal Text</option>
          <option value="title">Title</option>
          <option value="heading1">Heading 1</option>
          <option value="heading2">Heading 2</option>
          <option value="heading3">Heading 3</option>
        </select>
      </div>

      <!-- Font Size -->
      <div class="toolbar-group">
        <select @change="setFontSize($event.target.value)">
          <option value="12px">12px</option>
          <option value="14px">14px</option>
          <option value="16px">16px</option>
          <option value="18px">18px</option>
          <option value="20px">20px</option>
          <option value="24px">24px</option>
          <option value="28px">28px</option>
          <option value="32px">32px</option>
        </select>
      </div>

      <!-- Text Alignment -->
      <div class="toolbar-group">
        <button @click="activeEditor.chain().focus().setTextAlign('left').run()">Left</button>
        <button @click="activeEditor.chain().focus().setTextAlign('center').run()">Center</button>
        <button @click="activeEditor.chain().focus().setTextAlign('right').run()">Right</button>
      </div>

      <!-- Text Color -->
      <div class="toolbar-group">
        <label for="text-color">Text Color:</label>
        <input id="text-color" type="color" @input="changeTextColor($event.target.value)" />
      </div>

      <!-- Lists -->
      <div class="toolbar-group">
        <button :class="{ 'active': activeEditor.isActive('bulletList') }"
          @click="activeEditor.chain().focus().toggleBulletList().run()">Bullet List</button>
        <button :class="{ 'active': activeEditor.isActive('orderedList') }"
          @click="activeEditor.chain().focus().toggleOrderedList().run()">Ordered List</button>
      </div>

      <!-- Links -->
      <div class="toolbar-group">
        <button @click="insertLink">Insert Link</button>
        <button @click="removeLink" :disabled="!activeEditor.isActive('link')">Remove Link</button>
      </div>
    </div>
    <!-- Editor -->
    <div class="editor-content">
      <editor-content v-if="activeEditor" :editor="activeEditor" />
    </div>
  </div>
</template>

<style>
.editor-container {
  width: 100%;
  height: 100vh;
  display: flex;
  flex-direction: column;
  position: fixed;
  top: 0;
  left: 0;

}

.tab-bar {
  width: 100%;
  display: flex;
  background: #484848;
  padding: 4px 4px 0;
}

.tab {
  display: flex;
  align-items: center;
  padding: 8px 16px;
  background: #353535;
  /* border: 1px solid #fd0c0c; */
  border-bottom: none;
  border-radius: 12px 12px 0 0;
  margin-right: 4px;
  cursor: pointer;
}

.tab.active {
  background: #1e1e1e;
  /* border-color: #666; */
}

.close-tab {
  padding: 0px;
  background: transparent;
  border: none;
  color: #fff;
  cursor: pointer;
  margin-left: 8px;
}

.close-tab:disabled {
  opacity: 0.3;
  cursor: not-allowed;
}

.tab input {
  border: none;
  outline: none;
  background: transparent;
  margin-right: 8px;
  color: #fff;
}

.new-tab {
  /* padding: 6px; */
  height: 32px;
  width: 32px;
  align-items: center;
  /* border: 1px solid #ccc; */
  background: #353535;
  border-radius: 50%;
  cursor: pointer;
}

.toolbar {
  width: 100%;
  display: flex;
  padding: 8px;
  background: #363733;
  /* border-bottom: 1px solid #ddd; */
  flex-wrap: wrap;
  gap: 4px;
}

.toolbar-group {
  display: flex;
  margin-right: 10px;
  align-items: center;
}

.toolbar button,
.toolbar select,
.toolbar input {
  margin-right: 4px;
  padding: 4px 8px;
}

.toolbar button.active {
  background: #e0e0e0;
  color: #000;
}

.editor-content {
  flex: 1;
  overflow-y: scroll;
  overflow-x: hidden;

  /* min-height: 100%; */
}

.ProseMirror {
  width: 100%;
  height: 89vh;

  background-color: #121212;
  color: #fff;
  padding: 1.6rem 2.8rem;
  outline: none;
}

.ProseMirror:focus {
  outline: none;
}

.ProseMirror a {
  color: #4a9df6;
  text-decoration: underline;
  cursor: pointer;
}

.ProseMirror p {
  margin: 0.5em 0;
}

/* Apply styling to headings */
.ProseMirror h1 {
  font-size: 24px;
  margin: 1em 0 0.5em;
}

.ProseMirror h2 {
  font-size: 20px;
  margin: 0.8em 0 0.4em;
}

.ProseMirror h3 {
  font-size: 18px;
  margin: 0.6em 0 0.3em;
}

.codeBlock {
  background: #2d2d2d;
  color: #f8f8f2;
  padding: 1em;
  border-radius: 4px;
  overflow-x: auto;
}
</style>