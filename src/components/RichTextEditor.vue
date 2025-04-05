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
      // Create a default tab if no saved tabs exist
      addTab()
    }
  } catch (error) {
    console.error('Error loading tabs from storage:', error)
    // Create a default tab as fallback
    addTab()
  }
})

const addTab = () => {
  // Save current tab content before creating a new one
  saveCurrentTabContent()
  
  const newTab = {
    id: Date.now(),
    title: `Document ${tabs.length + 1}`,
    content: '',
  }
  tabs.push(newTab)
  activeTab.value = newTab.id
  saveTabs()
}

const closeTab = (tabId) => {
  // Save the content of the current tab before closing
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
  
  // Save current tab content before switching
  saveCurrentTabContent()
  
  // Destroy current editor instance to avoid memory leaks
  if (activeEditor.value) {
    activeEditor.value.destroy()
    activeEditor.value = null
  }

  activeTab.value = tabId
  
  // Wait for DOM update before creating new editor
  await nextTick()
  initializeEditor()
  
  // Save the updated tabs to storage
  saveTabs()
}

const initializeEditor = () => {
  if (!activeTab.value) return
  
  const currentTab = tabs.find(tab => tab.id === activeTab.value)
  if (!currentTab) return
  
  activeEditor.value = new Editor({
    content: currentTab.content,
    extensions: [
      StarterKit,
      Underline,
      Strike,
      Heading.configure({ levels: [1, 2] }),
      TextAlign.configure({ types: ['paragraph', 'heading'] }),
      Placeholder.configure({ placeholder: 'Start typing...' }),
      CharacterCount,
    ],
    onUpdate: ({ editor }) => {
      // On every update, save the content to the current tab
      currentTab.content = editor.getHTML()
      saveTabs()
    },
  })
}

const saveTabs = async () => {
  try {
    // Create a clean copy of the tabs data for storage
    const tabsToSave = tabs.map(tab => ({
      id: tab.id,
      title: tab.title,
      content: tab.content,
    }))
    
    await localforage.setItem(tabsStorageKey, tabsToSave)
  } catch (error) {
    console.error('Error saving tabs:', error)
  }
}

const changeFontFamily = (event) => {
  if (!activeEditor.value) return
  activeEditor.value.chain().focus().setMark('fontFamily', event.target.value).run()
}

const changeFontSize = (event) => {
  if (!activeEditor.value) return
  activeEditor.value.chain().focus().setMark('fontSize', event.target.value).run()
}

const changeTextColor = (event) => {
  if (!activeEditor.value) return
  activeEditor.value.chain().focus().setMark('color', event.target.value).run()
}

const changeBackgroundColor = (event) => {
  if (!activeEditor.value) return
  activeEditor.value.chain().focus().setMark('backgroundColor', event.target.value).run()
}

const insertLink = () => {
  if (!activeEditor.value) return
  const url = prompt('Enter URL:')
  if (url) {
    activeEditor.value.chain().focus().setLink({ href: url }).run()
  }
}

// Use watch to handle tab changes
watch(activeTab, (newTabId, oldTabId) => {
  if (newTabId && newTabId !== oldTabId) {
    // When activeTab changes, initialize the editor with the new tab's content
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

// Add window unload handler to save content before page close
window.addEventListener('beforeunload', () => {
  saveCurrentTabContent()
  saveTabs()
})
console.log('localforage:', localforage.getItem(tabsStorageKey))


</script>

<template>
  <div class="editor-container">
    <div class="tab-bar">
      <div v-for="tab in tabs" :key="tab.id" 
           :class="['tab', { 'active': activeTab === tab.id }]"
           @click="switchTab(tab.id)">
        <input v-model="tab.title" @click.stop @change="saveTabs" />
        <button @click.stop="closeTab(tab.id)" class="close-tab" 
                :disabled="tabs.length === 1">
          &times;
        </button>
      </div>
      <button class="new-tab" @click="addTab">+</button>
    </div>

    <div v-if="activeEditor" class="toolbar">
      <!-- Text Formatting -->
      <div class="toolbar-group">
        <button 
          class="toolbar-button" 
          @click="activeEditor.chain().focus().undo().run()"
          title="Undo"
        >
          ↩
        </button>
        <button 
          class="toolbar-button" 
          @click="activeEditor.chain().focus().redo().run()"
          title="Redo"
        >
          ↪
        </button>
      </div>
      <button :class="{ 'active': activeEditor.isActive('bold') }" @click="activeEditor.chain().focus().toggleBold().run()">Bold</button>
      
      <button :class="{ 'active': activeEditor.isActive('italic') }" @click="activeEditor.chain().focus().toggleItalic().run()">Italic</button>
      <button :class="{ 'active': activeEditor.isActive('underline') }" @click="activeEditor.chain().focus().toggleUnderline().run()">Underline</button>
      <button :class="{ 'active': activeEditor.isActive('strike') }" @click="activeEditor.chain().focus().toggleStrike().run()">Strikethrough</button>

      <!-- Text Alignment -->
      <button @click="activeEditor.chain().focus().setTextAlign('left').run()">Left</button>
      <button @click="activeEditor.chain().focus().setTextAlign('center').run()">Center</button>
      <button @click="activeEditor.chain().focus().setTextAlign('right').run()">Right</button>

      <!-- Font Size and Family -->
      <select @change="changeFontFamily($event)">
        <option value="Arial">Arial</option>
        <option value="Courier New">Courier New</option>
        <option value="Georgia">Georgia</option>
        <option value="Times New Roman">Times New Roman</option>
        <option value="Verdana">Verdana</option>
      </select>

      <select @change="changeFontSize($event)">
        <option value="12px">12px</option>
        <option value="14px">14px</option>
        <option value="16px">16px</option>
        <option value="18px">18px</option>
        <option value="20px">20px</option>
      </select>

      <!-- Text Color -->
      <input type="color" @input="changeTextColor($event)" />

      <!-- Background Color -->
      <input type="color" @input="changeBackgroundColor($event)" />

      <!-- Headings -->
      <select @change="activeEditor.chain().focus().toggleHeading({ level: $event.target.value }).run()">
        <option value="">Normal</option>
        <option value="1">Heading 1</option>
        <option value="2">Heading 2</option>
      </select>

      <!-- Lists -->
      <button :class="{ 'active': activeEditor.isActive('bulletList') }" @click="activeEditor.chain().focus().toggleBulletList().run()">Bullet List</button>
      <button :class="{ 'active': activeEditor.isActive('orderedList') }" @click="activeEditor.chain().focus().toggleOrderedList().run()">Ordered List</button>

      <!-- Links -->
      <button @click="insertLink">Insert Link</button>
    </div>

    <!-- Editor Content -->
    <div class="editor-content">
      <editor-content v-if="activeEditor" :editor="activeEditor" />
    </div>
  </div>
</template>

<style>
/* Updated styling */
.editor-container {
  width: 100%;
  height: 100vh;
  display: flex;
  flex-direction: column;
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  overflow: hidden;
  box-sizing: border-box;
}

.tab-bar {
  width: 100%;
  display: flex;
  background: #331717;
  padding: 4px 4px 0;
}

.tab {
  display: flex;
  align-items: center;
  padding: 8px 16px;
  background: #201c1c;
  border: 1px solid #fd0c0c;
  border-bottom: none;
  border-radius: 4px 4px 0 0;
  margin-right: 4px;
  cursor: pointer;
}

.tab.active {
  background: #5f5a5a;
  border-color: #666;
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
  padding: 0 12px;
  border: 1px solid #ccc;
  background: #bc5d5d;
  cursor: pointer;
}

.toolbar {
  width: 100%;
  display: flex;
  padding: 8px;
  background: #161313;
  border-bottom: 1px solid #ddd;
}

.toolbar button, .toolbar select, .toolbar input {
  margin-right: 8px;
  padding: 4px 8px;
}

.toolbar button.active {
  background: #e0e0e0;
}

.editor-content {
  flex: 1;
  /* padding: 16px; */
  /* overflow-y: auto; */
}

.ProseMirror {
  /* min-height: 100%; */
  width: 100vw;
  height: 100vh;
  outline: none;
  background-color: #201c1c;
  color: #fff;
  padding: 1.6rem 2.8rem;  
  outline: none;
}

.ProseMirror:focus {
  outline: none;
}
</style>