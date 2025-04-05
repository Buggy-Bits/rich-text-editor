<template>
  <div class="editor-container">
    <!-- Tabs Bar -->
    <div class="tabs-container">
      <div 
        v-for="(tab, index) in tabs" 
        :key="index"
        class="tab"
        :class="{ 'active-tab': activeTabIndex === index }"
        @click="switchTab(index)"
      >
        <span>{{ tab.name }}</span>
        <button 
          v-if="tabs.length > 1" 
          class="close-tab" 
          @click.stop="removeTab(index)"
        >
          &times;
        </button>
      </div>
      <button class="add-tab" @click="addTab">+</button>
    </div>

    <!-- Toolbar -->
    <div class="toolbar">
      <div class="toolbar-group">
        <button 
          class="toolbar-button" 
          @click="editor.chain().focus().toggleBold().run()"
          :class="{ 'is-active': editor.isActive('bold') }"
          title="Bold"
        >
          <span class="icon">B</span>
        </button>
        <button 
          class="toolbar-button" 
          @click="editor.chain().focus().toggleItalic().run()"
          :class="{ 'is-active': editor.isActive('italic') }"
          title="Italic"
        >
          <span class="icon">I</span>
        </button>
        <button 
          class="toolbar-button" 
          @click="editor.chain().focus().toggleUnderline().run()"
          :class="{ 'is-active': editor.isActive('underline') }"
          title="Underline"
        >
          <span class="icon">U</span>
        </button>
        <button 
          class="toolbar-button" 
          @click="editor.chain().focus().toggleStrike().run()"
          :class="{ 'is-active': editor.isActive('strike') }"
          title="Strike"
        >
          <span class="icon">S</span>
        </button>
      </div>

      <div class="toolbar-divider"></div>

      <div class="toolbar-group">
        <select 
          class="heading-select"
          @change="setHeading($event)" 
          :value="currentHeadingValue"
        >
          <option value="">Paragraph</option>
          <option value="1">Heading 1</option>
          <option value="2">Heading 2</option>
          <option value="3">Heading 3</option>
        </select>
      </div>

      <div class="toolbar-divider"></div>

      <div class="toolbar-group">
        <button 
          class="toolbar-button" 
          @click="editor.chain().focus().toggleBulletList().run()"
          :class="{ 'is-active': editor.isActive('bulletList') }"
          title="Bullet List"
        >
          • List
        </button>
        <button 
          class="toolbar-button" 
          @click="editor.chain().focus().toggleOrderedList().run()"
          :class="{ 'is-active': editor.isActive('orderedList') }"
          title="Numbered List"
        >
          1. List
        </button>
      </div>

      <div class="toolbar-divider"></div>

      <div class="toolbar-group">
        <button 
          class="toolbar-button" 
          @click="editor.chain().focus().toggleBlockquote().run()"
          :class="{ 'is-active': editor.isActive('blockquote') }"
          title="Quote"
        >
          "Quote"
        </button>
        <button 
          class="toolbar-button" 
          @click="editor.chain().focus().setHorizontalRule().run()"
          title="Horizontal Rule"
        >
          —
        </button>
      </div>

      <div class="toolbar-divider"></div>
      <div class="toolbar-group">
        <button 
          class="toolbar-button" 
          @click="editor.chain().focus().undo().run()"
          title="Undo"
        >
          ↩
        </button>
        <button 
          class="toolbar-button" 
          @click="editor.chain().focus().redo().run()"
          title="Redo"
        >
          ↪
        </button>
      </div>
    </div>

    <!-- Editor -->
    <div class="editor-content">
      <editor-content :editor="editor" />
    </div>
  </div>
</template>

<script>
import { Editor, EditorContent } from '@tiptap/vue-3'
import StarterKit from '@tiptap/starter-kit'
import Underline from '@tiptap/extension-underline'
import Placeholder from '@tiptap/extension-placeholder'
import localforage from 'localforage'
localforage.setItem('somekey', 'hello value').then(function (value) {
    console.log(value);
}).catch(function(err) {
    console.log(err);
});
localforage.getItem('somekey').then(function(value) {
    console.log(value);
}).catch(function(err) {
    console.log(err);
});
console.log('localforage is: ', localforage)
export default {
  name: 'TiptapEditor',
  components: {
    EditorContent,
  },
  data() {
    return {
      editor: null,
      tabs: [],
      activeTabIndex: 0,
      storageKey: 'tiptap-tabs-data',
      defaultTabContent: '<p>Start writing in this tab...</p>',
    }
  },
  computed: {
    currentHeadingValue() {
      if (this.editor.isActive('heading', { level: 1 })) return '1'
      if (this.editor.isActive('heading', { level: 2 })) return '2'
      if (this.editor.isActive('heading', { level: 3 })) return '3'
      return ''
    }
  },
  mounted() {
    this.loadFromStorage()
    
    if (!this.tabs.length) {
      this.tabs = [
        { 
          name: 'Document 1',
          content: this.defaultTabContent,
        }
      ]
    }
    
    this.initEditor()
  },
  beforeUnmount() {
    this.editor.destroy()
  },
  methods: {
    initEditor() {
      this.editor = new Editor({
        content: this.tabs[this.activeTabIndex].content,
        extensions: [
          StarterKit,
          Underline,
          Placeholder.configure({
            placeholder: 'Start writing...',
          }),
        ],
        onUpdate: ({ editor }) => {
          this.tabs[this.activeTabIndex].content = editor.getHTML()
          this.saveToStorage()
        },
      })
    },
    
    setHeading(event) {
      const level = event.target.value
      
      if (level === '') {
        this.editor.chain().focus().setParagraph().run()
      } else {
        this.editor.chain().focus().toggleHeading({ level: parseInt(level) }).run()
      }
    },
    
    switchTab(index) {
      if (this.activeTabIndex === index) return
      
      // Save current content first
      if (this.editor) {
        this.tabs[this.activeTabIndex].content = this.editor.getHTML()
      }
      
      this.activeTabIndex = index
      
      // Update editor content
      this.editor.commands.setContent(this.tabs[index].content)
      this.saveToStorage()
    },
    
    addTab() {
      const newTabName = `Document ${this.tabs.length + 1}`
      this.tabs.push({
        name: newTabName,
        content: this.defaultTabContent,
      })
      
      this.switchTab(this.tabs.length - 1)
      this.saveToStorage()
    },
    
    removeTab(index) {
      if (this.tabs.length <= 1) return
      
      this.tabs.splice(index, 1)
      
      // Adjust active tab if needed
      if (index <= this.activeTabIndex) {
        this.activeTabIndex = Math.max(0, this.activeTabIndex - 1)
        this.editor.commands.setContent(this.tabs[this.activeTabIndex].content)
      }
      
      this.saveToStorage()
    },
    
    saveToStorage() {
      const tabsData = this.tabs.map(tab => ({
        name: tab.name,
        content: tab.content
      }))
      
      localStorage.setItem(this.storageKey, JSON.stringify({
        tabs: tabsData,
        activeTabIndex: this.activeTabIndex
      }))
    },
    
    loadFromStorage() {
      const storedData = localStorage.getItem(this.storageKey)
      if (storedData) {
        const parsedData = JSON.parse(storedData)
        this.tabs = parsedData.tabs
        this.activeTabIndex = parsedData.activeTabIndex
      }
    }
  }
}
</script>

<style scoped>
.editor-container {
  display: flex;
  flex-direction: column;
  height: 100vh;
  width: 100%;
  max-width: 900px;
  margin: 0 auto;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, 
    Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
}

.tabs-container {
  display: flex;
  border-bottom: 1px solid #e0e0e0;
  background-color: #f9f9f9;
  padding: 8px 8px 0;
  overflow-x: auto;
}

.tab {
  padding: 8px 16px;
  border: 1px solid #e0e0e0;
  border-bottom: none;
  border-radius: 4px 4px 0 0;
  margin-right: 4px;
  background-color: #f0f0f0;
  cursor: pointer;
  display: flex;
  align-items: center;
  font-size: 14px;
  white-space: nowrap;
}

.active-tab {
  background-color: white;
  border-bottom: 1px solid white;
  margin-bottom: -1px;
  font-weight: 500;
}

.close-tab {
  margin-left: 8px;
  border: none;
  background: none;
  font-size: 16px;
  cursor: pointer;
  color: #666;
  padding: 0 4px;
}

.close-tab:hover {
  color: #f44336;
}

.add-tab {
  border: none;
  background: none;
  padding: 8px 12px;
  border-radius: 4px;
  cursor: pointer;
  font-size: 16px;
  font-weight: bold;
  color: #1a73e8;
}

.add-tab:hover {
  background-color: #e8f0fe;
}

/* Toolbar */
.toolbar {
  display: flex;
  padding: 8px 16px;
  border-bottom: 1px solid #e0e0e0;
  flex-wrap: wrap;
  gap: 4px;
  background-color: white;
}

.toolbar-group {
  display: flex;
  margin-right: 4px;
}

.toolbar-button {
  background: none;
  border: none;
  border-radius: 4px;
  padding: 6px 8px;
  cursor: pointer;
  font-size: 14px;
  color: #444;
}

.toolbar-button:hover {
  background-color: #f1f3f4;
}

.toolbar-button.is-active {
  background-color: #e8f0fe;
  color: #1a73e8;
}

.toolbar-divider {
  width: 1px;
  background-color: #e0e0e0;
  margin: 0 8px;
}

.heading-select {
  padding: 6px 8px;
  border-radius: 4px;
  border: 1px solid #e0e0e0;
  background-color: white;
  font-size: 14px;
}


.editor-content {
  flex-grow: 1;
  overflow-y: auto;
  padding: 16px 24px;
  background-color: white;
}

:deep(.ProseMirror) {
  outline: none;
  min-height: calc(100vh - 130px);
}

:deep(.ProseMirror p) {
  margin: 0.75em 0;
}

:deep(.ProseMirror h1) {
  font-size: 1.75em;
  margin: 1em 0 0.5em;
}

:deep(.ProseMirror h2) {
  font-size: 1.5em;
  margin: 1em 0 0.5em;
}

:deep(.ProseMirror h3) {
  font-size: 1.25em;
  margin: 1em 0 0.5em;
}

:deep(.ProseMirror blockquote) {
  border-left: 3px solid #ddd;
  margin-left: 0;
  margin-right: 0;
  padding-left: 1em;
  color: #666;
}

:deep(.ProseMirror ul) {
  padding-left: 1.5em;
}

:deep(.ProseMirror ol) {
  padding-left: 1.5em;
}

:deep(.ProseMirror hr) {
  border: none;
  border-top: 2px solid #e0e0e0;
  margin: 2em 0;
}


@media (max-width: 640px) {
  .toolbar {
    padding: 4px 8px;
  }
  
  .toolbar-button {
    padding: 6px;
  }
  
  .editor-content {
    padding: 12px 16px;
  }
}
</style>