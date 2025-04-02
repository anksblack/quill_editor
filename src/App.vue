<template>
  <div class="main-container">

    <div class="editor-container">
      <div class="editor-header">
        <button @click="exportToPdf" class="export-btn pdf-btn">
          <svg class="icon" viewBox="0 0 24 24">
            <path d="M14,9H19.5L14,3.5V9M7,2H15L21,8V20A2,2 0 0,1 19,22H7C5.89,22 5,21.1 5,20V4A2,2 0 0,1 7,2M7,20H19V16H7V20M17,17V19H7V17H17M17,12.5V14H7V12.5H17Z"/>
          </svg>
          Export to PDF
        </button>
        <button @click="exportToDoc" class="export-btn doc-btn">
          <svg class="icon" viewBox="0 0 24 24">
            <path d="M14,2H6C4.89,2 4,2.89 4,4V20A2,2 0 0,0 6,22H18A2,2 0 0,0 20,20V8L14,2M18,20H6V4H13V9H18V20M8,15H16V17H8V15M8,11H10V13H8V11M8,7H10V9H8V7M12,7H16V9H12V7M12,11H16V13H12V11Z"/>
          </svg>
          Export to Document
        </button>
      </div>
      <QuillEditor
        ref="quillEditor"
        v-model:content="editorContent"
        contentType="html"
        :toolbar="toolbarOptions"
        theme="snow"
      />
    </div>

    <!-- Transfer buttons -->
    <div class="transfer-buttons">
      <button @click="transferToEditor">↩</button>
      <button @click="transferFromEditor">➔</button>
    </div>

    <!-- External content -->
    <div class="external-content">
      <!-- Example external content with various formatting -->
      <h1>Black Hole:</h1>
      <h4>
        Black holes have two parts. There is the event horizon, which you can think of as the
        surface, though it's simply the point where the gravity gets too strong for anything to escape.
        And then, at the center, is the singularity.
      </h4>
      <h2>Points About Black Holes</h2>
      <div>
        <p>• A black hole is a region in space where gravity is so strong that nothing, not even light, can escape it.</p>
        <p>• The boundary around a black hole is called the event horizon.</p>
      </div>
      
      <center><button @click="addCitations">Add Citations</button></center>
      <div class="external-widget">
        <div class="card">
          Select text in this card and transfer to editor
        </div>
      </div>
      <textarea
        ref="externalTextBox"
        class="external-textbox"
        placeholder="Or select text here"
      ></textarea>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import { QuillEditor } from '@vueup/vue-quill'
import '@vueup/vue-quill/dist/vue-quill.snow.css'
import jsPDF from 'jspdf'
import html2canvas from 'html2canvas'

const editorContent = ref('')
const externalTextBox = ref(null)
const quillEditor = ref(null)

const toolbarOptions = [
  ['bold', 'italic', 'underline', 'strike'],
  ['blockquote', 'code-block'],
  [{ header: 1 }, { header: 2 }],
  [{ list: 'ordered' }, { list: 'bullet' }],
  [{ script: 'sub' }, { script: 'super' }],
  [{ indent: '-1' }, { indent: '+1' }],
  [{ direction: 'rtl' }],
  [{ size: ['small', false, 'large', 'huge'] }],
  [{ header: [1, 2, 3, 4, 5, 6, false] }],
  [{ color: [] }, { background: [] }],
  [{ font: [] }],
  [{ align: [] }],
  ['clean'],
  ['link', 'image', 'video']
]

// Helper: Get Quill instance.
const getEditor = () => quillEditor.value.getQuill()

// Transfer external selection into Quill while preserving formatting.
const transferToEditor = () => {
  const selection = window.getSelection()
  if (!selection.toString().trim()) {
    alert('Please select some text outside the editor')
    return
  }
  
  let htmlToInsert = ''
  for (let i = 0; i < selection.rangeCount; i++) {
    const range = selection.getRangeAt(i)
    let preserveTag = ''
    const anchorEl = range.startContainer.nodeType === Node.TEXT_NODE
      ? range.startContainer.parentElement
      : range.startContainer
    if (anchorEl && /^H[1-6]$/i.test(anchorEl.tagName)) {
      const elText = anchorEl.innerText.trim()
      if (range.toString().trim() === elText) {
        preserveTag = anchorEl.outerHTML
      }
    }
    if (preserveTag) {
      htmlToInsert += preserveTag
    } else {
      const fragment = range.cloneContents()
      const tempDiv = document.createElement('div')
      tempDiv.appendChild(fragment)
      htmlToInsert += tempDiv.innerHTML
    }
  }

  const editor = getEditor()
  const rangeIndex = editor.getSelection(true)?.index || editor.getLength()
  editor.clipboard.dangerouslyPasteHTML(rangeIndex, htmlToInsert)

  setTimeout(() => {
    const newIndex = rangeIndex + htmlToInsert.length
    editor.setSelection(newIndex, 0)
  }, 50)

  console.log("EDITOR CONTENT",editorContent.value)
}

// Transfer selected text from Quill to external text box.
const transferFromEditor = () => {
  const editor = getEditor()
  const selection = editor.getSelection()
  if (!selection || selection.length === 0) {
    alert('Please select text in the editor first')
    return
  }
  const selectedContent = editor.getContents(selection.index, selection.length)
  const plainText = selectedContent.ops.map(op => op.insert).join('')

  const textarea = externalTextBox.value
  const startPos = textarea.selectionStart
  const endPos = textarea.selectionEnd
  textarea.value =
    textarea.value.substring(0, startPos) +
    plainText +
    textarea.value.substring(endPos)
  textarea.selectionStart = textarea.selectionEnd = startPos + plainText.length
  textarea.focus()
  console.log("TEXT AREA VALUE",textarea.value)
}

// Add citations: insert "Citations" heading (only once) and append citation links (as paragraphs).
const addCitations = () => {
  const editor = getEditor()
  const contentText = editor.getText()
  if (!contentText.includes("\nCitations\n")) {
    editor.insertText(editor.getLength(), "\nCitations\n", { header: 2 })
  }
  
  const randomDocId = Math.random().toString(36).substring(2, 10)
  const randomLink = `https://example.com/pdf?doc=${randomDocId}`
  const citationHtml = `<p><a href="${randomLink}" target="_blank">Download PDF</a></p>`
  
  const insertPosition = editor.getLength()
  editor.clipboard.dangerouslyPasteHTML(insertPosition, citationHtml)
}

// Helper function to remove citation links (anchor tags with "Download PDF" text)
const removeCitationLinks = (container) => {
  const links = container.querySelectorAll('a');
  links.forEach(link => {
    if (link.textContent.trim() === "Download PDF") {
      // Remove the entire link element.
      link.remove();
    }
  });
}

// Export editor content to PDF using html2canvas and jsPDF.
const exportToPdf = async () => {
  const editor = getEditor();
  // Clone the editor's content.
  const clonedContent = editor.root.cloneNode(true);
  // Remove citation links from the clone.
  removeCitationLinks(clonedContent);

  // Create a temporary container off-screen
  const tempDiv = document.createElement('div');
  tempDiv.style.position = 'fixed';
  tempDiv.style.top = '-10000px';
  tempDiv.style.left = '-10000px';
  tempDiv.appendChild(clonedContent);
  document.body.appendChild(tempDiv);

  try {
    // Render the temporary container with html2canvas.
    const canvas = await html2canvas(clonedContent, {
      useCORS: true,
      allowTaint: true,
      scrollY: -window.scrollY
    });
    const imgData = canvas.toDataURL('image/png');
    
    const pdf = new jsPDF({
      orientation: 'portrait',
      unit: 'pt',
      format: 'a4'
    });
    
    const pageWidth = pdf.internal.pageSize.getWidth();
    const imgHeight = (canvas.height * pageWidth) / canvas.width;
    
    pdf.addImage(imgData, 'PNG', 0, 0, pageWidth, imgHeight);
    pdf.save('document.pdf');
  } catch (error) {
    console.error('Error exporting to PDF:', error);
  } finally {
    // Clean up the temporary container.
    document.body.removeChild(tempDiv);
  }
}

// Export editor content to a Word document.
const exportToDoc = () => {
  const editor = getEditor();
  // Clone the editor's content.
  const clonedContent = editor.root.cloneNode(true);
  // Remove citation links from the cloned content.
  removeCitationLinks(clonedContent);
  
  const contentHtml = clonedContent.innerHTML;
  const preHtml = "<html><head><meta charset='utf-8'><title>Export</title></head><body>";
  const postHtml = "</body></html>";
  const html = preHtml + contentHtml + postHtml;
  
  const blob = new Blob(['\ufeff', html], {
    type: 'application/msword'
  });
  const url = URL.createObjectURL(blob);
  
  const a = document.createElement('a');
  a.href = url;
  a.download = 'document.doc';
  document.body.appendChild(a);
  a.click();
  document.body.removeChild(a);
}

</script>


<style>
/* Base Styles */
body {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  background: #f0f2f5;
  margin: 0;
  padding: 20px;
}

.main-container {
  display: grid;
  grid-template-columns: 1fr auto 1fr;
  gap: 30px;
  max-width: 1400px;
  margin: 0 auto;
  background: white;
  padding: 30px;
  border-radius: 12px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}




.editor-container {
  height: 700px; /* Fixed height */
  display: flex;
  flex-direction: column;
  border: 2px solid #e0e0e0;
  border-radius: 10px;
  background: white;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
}

.ql-toolbar {
  position: sticky !important;
  top: 0;
  background: white !important;
  z-index: 100;
  border-radius: 8px 8px 0 0 !important;
}

.ql-container {
  flex: 1;
  overflow-y: auto;
  border: none !important;
  border-radius: 0 0 8px 8px !important;
}

.ql-editor {
  min-height: 100% !important;
  padding: 20px;
  overflow-wrap: break-word;
  word-break: break-word;
  white-space: pre-wrap;
  box-sizing: border-box;
}
















.editor-header {
  padding: 10px;
  background: white;
  border-bottom: 1px solid #e0e0e0;
  z-index: 100;
  position: relative;
}

.editor-container button {
  margin: 10px;
  padding: 8px 16px;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.3s ease;
  font-weight: 500;
}

.editor-container button:hover {
  transform: translateY(-1px);
  box-shadow: 0 2px 8px rgba(102, 126, 234, 0.4);
}

/* Transfer Buttons */
.transfer-buttons {
  display: flex;
  flex-direction: column;
  gap: 15px;
  align-self: center;
}

.transfer-buttons button {
  padding: 14px 24px;
  background: linear-gradient(135deg, #4f46e5 0%, #9333ea 100%);
  color: white;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.3s ease;
  font-size: 18px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.transfer-buttons button:hover {
  transform: scale(1.05);
  box-shadow: 0 4px 8px rgba(79, 70, 229, 0.3);
}

/* External Content Section */
.external-content {
  display: flex;
  flex-direction: column;
  gap: 20px;
  background: #f8f9fa;
  padding: 20px;
  border-radius: 10px;
  border: 2px solid #e0e0e0;
}

.external-content h1 {
  color: #1a237e;
  margin: 0;
  font-size: 28px;
  border-bottom: 2px solid #4f46e5;
  padding-bottom: 8px;
}

.external-content h2 {
  color: #2d3748;
  margin: 15px 0 10px;
  font-size: 22px;
}

.external-content h4 {
  color: #4a5568;
  line-height: 1.7;
  margin: 10px 0;
  font-weight: 400;
}

.external-content p {
  padding: 6px;
  border-radius: 6px;
  margin: 8px 0;
}

.external-widget .card {
  padding: 20px;
  border: 2px dashed #c7d2fe;
  border-radius: 8px;
  background: white;
  cursor: text;
  color: #4f46e5;
  text-align: center;
  transition: all 0.2s ease;
}

.external-widget .card:hover {
  border-color: #4f46e5;
  background: #f8f9ff;
}

.external-textbox {
  width: 95%;
  height: 120px;
  padding: 15px;
  border: 2px solid #e0e0e0;
  border-radius: 8px;
  resize: vertical;
  font-size: 14px;
  line-height: 1.6;
  transition: border-color 0.3s ease;
}

.external-textbox:focus {
  outline: none;
  border-color: #4f46e5;
  box-shadow: 0 0 0 3px rgba(79, 70, 229, 0.1);
}

/* Citations Button */
.external-content button {
  align-self: flex-start;
  padding: 10px 20px;
  background: linear-gradient(135deg, #10b981 0%, #059669 100%);
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.3s ease;
  font-weight: 500;
  margin-top: 10px;
}

.external-content button:hover {
  transform: translateY(-1px);
  box-shadow: 0 2px 8px rgba(16, 185, 129, 0.4);
}

/* Responsive Design */
@media (max-width: 1200px) {
  .main-container {
    grid-template-columns: 1fr;
    grid-template-rows: auto auto auto;
  }
  
  .transfer-buttons {
    flex-direction: row;
    justify-content: center;
  }
  
  .editor-container,
  .external-content {
    width: auto;
    max-width: 100%;
  }
}

.pdf-btn .icon path {
  fill: #dc2626; /* Red color for PDF icon */
}

.doc-btn .icon path {
  fill: #2563eb; /* Blue color for Document icon */
}

/* Update your button styles */
.editor-header button {
  margin: 8px;
  padding: 8px 14px;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-weight: 500;
  display: inline-flex;
  align-items: center;
  gap: 8px;
  transition: all 0.3s ease;
}

/* Optional: Add hover effects specific to icon colors */
.pdf-btn:hover .icon path {
  fill: #f87171; /* Lighter red on hover */
}

.doc-btn:hover .icon path {
  fill: #60a5fa; /* Lighter blue on hover */
}

/* Keep existing icon sizing */
.icon {
  width: 20px;
  height: 20px;
  transition: fill 0.3s ease;
}

</style>