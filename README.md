# My Notes editor

A personal rich text editor built with Vue.js and TipTap to manage temperory notes, todo lists, and important links in a dark-themed interface.

## Project Overview
This is a personal project I developed to help manage my notes and tasks while learning software development. As a college student spending significant time on my laptop, I needed a centralized place to store todos, important links, and notes for my courses.

While there are many note-taking applications available, I wanted to build my own solution that:
- Simple and fast
- Build for my specific needs
- Works with a dark theme allows me to work night time also
- Supports rich text formatting for simple butification
- Allows for multiple document tabs as i have to manage lot of things
- Persists data between sessions 

## Features

- **Multi-tab Document Management**: Create, rename, and switch between multiple documents
- **Rich Text Editing**:
  - Basic formatting (bold, italic, underline, strikethrough)
  - Headings and text styles
  - Font size adjustment
  - Text alignment
  - Text colouring
  - Simple bullet lists and ordered lists
  - Code blocks for code snippets
  - Link management (redirect to link)
- **Persistent Storage**: All documents are automatically saved using localforage
- **Custom Undo/Redo**: Implementation of custom history management
- **Keyboard Shortcuts**: Ctrl + Z support for undo operations
- **Dark Theme**: Designed with a dark color scheme for reduced eye strain

## Technologies Used

- **Vue.js**: Front-end framework
- **TipTap**: Text editor framework for Vue
- **LocalForage**: Client-side storage library

## Installation

```bash
# Clone the repository
git clone https://github.com/Buggy-Bits/rich-text-editor.git

# Navigate to project directory
cd rich-text-editor

# Install dependencies
npm install

# Run the dev server - will be live at localhost:5173
npm run dev
```


## Learning Outcomes

This project allowed me to:
- Gain some experience with Vue.js composition API
- Learn how to implement a rich text editor using TipTap
- Understand client-side storage with LocalForage
- Practice state management in Vue components
- Implement custom undo/redo functionality
