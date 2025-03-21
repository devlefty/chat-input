<template>
  <div class="template-input-container">
    <div 
      ref="editorRef" 
      class="template-editor" 
      role="textbox" 
      aria-multiline="true" 
      contenteditable="false" 
      @click="handleEditorClick"
      @keydown="handleGlobalKeydown"
    >
      <!-- 模板内容渲染 -->
      <template v-for="(part, index) in templateParts" :key="index">
        <!-- 普通文本 -->
        <span 
          v-if="part.type === 'text'" 
          class="text-content" 
          contenteditable="true"
          :data-part-index="index"
        >{{ part.content }}</span>

        <!-- 可填写区域 -->
        <span
          v-else-if="part.type === 'placeholder'"
          class="placeholder-wrapper"
          :class="{ 'has-content': part.value }"
          @click.stop="focusPlaceholder(index)"
          contenteditable="false"
        >
          <!-- 用于撑开元素宽度和高度的文本 -->
          <span :class="part.value ?  'placeholder-sizer-hidden':'placeholder-sizer'">[{{ part.label }}]</span>

          <!-- 显示的占位符文本 -->
          <span v-if="!part.value" class="placeholder-text">{{ part.label }}</span>

          <!-- 可编辑区域 -->
          <span
            :ref="el => { if(el) editableFields[index] = el }"
            :data-index="index"
            :data-part-index="index"
            contenteditable="true"
            class="editable-content"
            @blur="onFieldBlur(index)"
            @input="(e) => updateFieldValue(index, e)"
            @keydown="(e) => onKeyDown(e, index)"
          >{{ part.value }}</span>
        </span>
      </template>
    </div>

    <!-- 完整内容预览和提交按钮 -->
    <div class="preview-container" v-if="showPreview">
      <div class="preview-title">生成的提示:</div>
      <div class="preview-content">{{ generatedContent }}</div>
    </div>

    <button class="submit-button" @click="submitTemplate">提交</button>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, nextTick } from 'vue';

// 定义模板结构
const templateStructure = [
  { type: 'text', content: '我是一名' },
  { type: 'placeholder', label: '输入职业', value: '财神' },
  { type: 'text', content: '，帮我就' },
  { type: 'placeholder', label: '主题', value: '' },
  { type: 'text', content: '进行头脑风暴，不少于 ' },
  { type: 'placeholder', label: '输入数字', value: '' },
  { type: 'text', content: ' 个点子，每个创新想法都不能和其他重复，需要有明显的差异性。' }
];

// 组件状态
const templateParts = ref(JSON.parse(JSON.stringify(templateStructure)));
const editorRef = ref(null);
const editableFields = ref([]); // 将作为对象数组使用
const showPreview = ref(false);

// 计算生成的完整内容
const generatedContent = computed(() => {
  return templateParts.value.map(part => {
    if (part.type === 'text') return part.content;
    return part.value || `[${part.label}]`;
  }).join('');
});

// 检查所有字段是否已填写
const isTemplateComplete = computed(() => {
  return !templateParts.value.some(part => 
    part.type === 'placeholder' && !part.value.trim()
  );
});

// 点击编辑区域处理
const handleEditorClick = () => {
  // 当点击整个编辑区域时，不执行特殊操作
};

// 完全重写键盘导航逻辑
const handleKeyNavigation = (event, currentIndex) => {
  console.log('键盘导航触发', event.key, currentIndex);
  // 获取当前输入元素
  const currentElement = event.target;
  
  // 检查光标位置
  const selection = window.getSelection();
  const isAtStart = selection.anchorOffset === 0;
  const isAtEnd = currentElement.textContent && 
                  selection.anchorOffset === currentElement.textContent.length;
  
  // 查找所有placeholder索引
  const placeholderIndices = [];
  templateParts.value.forEach((part, idx) => {
    if (part.type === 'placeholder') {
      placeholderIndices.push(idx);
    }
  });
  
  // 当前placeholder在列表中的位置
  const currentPosition = placeholderIndices.indexOf(currentIndex);
  
  // 处理键盘事件
  if (event.key === 'ArrowLeft' && isAtStart) {
    // 向左移动且已在文本开头
    if (currentPosition > 0) {
      event.preventDefault(); // 阻止默认行为
      const prevIndex = placeholderIndices[currentPosition - 1];
      focusEditableField(prevIndex, 'end');
      return;
    }
  } 
  else if (event.key === 'ArrowRight' && isAtEnd) {
    // 向右移动且已在文本末尾
    if (currentPosition < placeholderIndices.length - 1) {
      event.preventDefault(); // 阻止默认行为
      const nextIndex = placeholderIndices[currentPosition + 1];
      focusEditableField(nextIndex, 'start');
      return;
    }
  }
  else if (event.key === 'Tab') {
    // Tab键导航
    event.preventDefault(); // 阻止默认的Tab行为
    
    if (event.shiftKey) {
      // Shift+Tab: 上一个字段
      if (currentPosition > 0) {
        const prevIndex = placeholderIndices[currentPosition - 1];
        focusEditableField(prevIndex, 'end');
      }
    } else {
      // Tab: 下一个字段
      if (currentPosition < placeholderIndices.length - 1) {
        const nextIndex = placeholderIndices[currentPosition + 1];
        focusEditableField(nextIndex, 'start');
      }
    }
  }
};


// 更新字段值
const updateFieldValue = (index, event) => {
  templateParts.value[index].value = event.target.textContent;
  showPreview.value = isTemplateComplete.value;
};

// 失去焦点时的处理
const onFieldBlur = (index) => {
  // 如果需要在失焦时执行额外逻辑，可以在这里添加
};

// 添加直接的keydown处理程序
const onKeyDown = (event, index) => {
  console.log('键盘事件触发:', event.key, index);
  handleKeyNavigation(event, index);
};

// 提交模板内容
const submitTemplate = () => {
  if (!isTemplateComplete.value) {
    alert('请填写所有必填字段');
    return;
  }

  // 这里可以emit事件将内容发送给父组件
  const result = generatedContent.value;
  console.log('提交的模板内容:', result);

  // 向父组件发送生成的内容
  emit('submit', result);
};

// 定义组件对外暴露的事件
const emit = defineEmits(['submit']);

// 组件挂载后的初始化
onMounted(async () => {
  // 确保DOM已更新
  await nextTick();
});

// 添加全局键盘事件处理，处理所有编辑区域
const handleGlobalKeydown = (event) => {
  // 获取当前活动元素
  const activeElement = document.activeElement;
  
  // 判断活动元素是否是我们的可编辑区域
  const isTextContent = activeElement.classList.contains('text-content');
  const isEditableContent = activeElement.classList.contains('editable-content');
  
  // 如果不是可编辑区域，则不处理
  if (!isTextContent && !isEditableContent) return;
  
  console.log('全局键盘事件:', event.key);
  
  // 获取当前元素的索引
  const partIndex = parseInt(activeElement.dataset.partIndex);
  
  // 检查光标位置
  const selection = window.getSelection();
  const isAtStart = selection.anchorOffset === 0;
  const isAtEnd = activeElement.textContent && 
                  selection.anchorOffset === activeElement.textContent.length;
  
  // 键盘导航逻辑
  if (event.key === 'ArrowLeft' && isAtStart) {
    // 向左移动，查找前一个可编辑元素
    navigateToPreviousElement(partIndex);
    event.preventDefault();
  }
  else if (event.key === 'ArrowRight' && isAtEnd) {
    // 向右移动，查找后一个可编辑元素
    navigateToNextElement(partIndex);
    event.preventDefault();
  }
  else if (event.key === 'Tab') {
    // Tab键导航
    event.preventDefault();
    if (event.shiftKey) {
      // Shift+Tab: 上一个可编辑元素
      navigateToPreviousElement(partIndex);
    } else {
      // Tab: 下一个可编辑元素
      navigateToNextElement(partIndex);
    }
  }
};

// 导航到前一个可编辑元素
const navigateToPreviousElement = (currentIndex) => {
  // 寻找前一个可编辑元素
  for (let i = currentIndex - 1; i >= 0; i--) {
    const part = templateParts.value[i];
    if (part.type === 'text') {
      // 找到前一个文本区域
      focusTextElement(i, 'end');
      return;
    } else if (part.type === 'placeholder') {
      // 找到前一个占位符
      focusPlaceholder(i, 'end');
      return;
    }
  }
};

// 导航到后一个可编辑元素
const navigateToNextElement = (currentIndex) => {
  // 寻找后一个可编辑元素
  for (let i = currentIndex + 1; i < templateParts.value.length; i++) {
    const part = templateParts.value[i];
    if (part.type === 'text') {
      // 找到后一个文本区域
      focusTextElement(i, 'start');
      return;
    } else if (part.type === 'placeholder') {
      // 找到后一个占位符
      focusPlaceholder(i, 'start');
      return;
    }
  }
};

// 聚焦普通文本元素
const focusTextElement = async (index, position = 'start') => {
  await nextTick();
  
  // 获取所有文本区域
  const textElements = document.querySelectorAll('.text-content');
  
  for (const element of textElements) {
    if (parseInt(element.dataset.partIndex) === index) {
      element.focus();
      
      // 设置光标位置
      const range = document.createRange();
      const selection = window.getSelection();
      
      if (position === 'end' && element.lastChild) {
        range.setStartAfter(element.lastChild);
      } else {
        range.setStart(element, 0);
      }
      
      range.collapse(true);
      selection.removeAllRanges();
      selection.addRange(range);
      
      break;
    }
  }
};

// 保留原来的focusEditableField函数，重命名为focusEditableField
const focusEditableField = async (index, position = 'start') => {
  await nextTick();
  
  // 获取所有可编辑字段元素
  const fields = document.querySelectorAll('.editable-content');
  
  // 找到目标字段
  let targetField = null;
  for (const field of fields) {
    if (parseInt(field.dataset.index) === index) {
      targetField = field;
      break;
    }
  }
  
  if (targetField) {
    // 聚焦目标字段
    targetField.focus();
    
    // 设置光标位置
    const range = document.createRange();
    const sel = window.getSelection();
    
    try {
      if (position === 'end') {
        if (targetField.lastChild) {
          range.setStartAfter(targetField.lastChild);
        } else {
          range.setStart(targetField, 0);
        }
      } else {
        if (targetField.firstChild) {
          range.setStartBefore(targetField.firstChild);
        } else {
          range.setStart(targetField, 0);
        }
      }
      
      range.collapse(true);
      sel.removeAllRanges();
      sel.addRange(range);
    } catch (e) {
      console.error('设置光标位置失败:', e);
    }
  }
};

// 更新focusPlaceholder函数
const focusPlaceholder = (index, position = 'start') => {
  focusEditableField(index, position);
};
</script>

<style scoped>
.template-input-container {
  font-family: system-ui, -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  max-width: 800px;
  margin: 0 auto;
  padding: 16px;
  font-size: 16px;
  text-align: left;
}

.template-editor {
  position: relative;
  white-space: pre-wrap;
  overflow-wrap: break-word;
  border: 1px solid #ddd;
  border-radius: 8px;
  padding: 12px 16px;
  min-height: 100px;
  background-color: #fff;
  cursor: text;
  line-height: 1.5;
}

.text-content {
  display: inline;
  outline: none;
}

/* 占位符包装器 */
.placeholder-wrapper {
  display: inline-flex;
  position: relative;
  border-radius: 4px;
  background-color: rgba(0, 120, 255, 0.05);
  padding: 2px 8px;
  margin: 0 2px;
  cursor: pointer;
  transition: background-color 0.2s, border-color 0.2s;
}

.placeholder-wrapper.has-content {
  background-color: rgba(0, 120, 255, 0.1);
  /* border: 1px solid #0078ff; */
}

/* 用于撑开元素宽度和高度的文本 */
.placeholder-sizer {
  visibility: hidden;
  white-space: pre;
  display: block;
  line-height: 1.6; /* 增加行高改善间距 */
  opacity: 1;
  pointer-events: none;
  height: auto;
}
.placeholder-sizer-hidden {
  display: none;
}

/* 占位符文本 - 显示在占位符中 */
.placeholder-text {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  color: #0078ff;
  opacity: 0.8;
  pointer-events: none;
  white-space: nowrap;
}

/* 可编辑内容 */
.editable-content {
  position: absolute;
  top: 0;
  right: 0;
  bottom: 0px !important;
  left: 0 !important;
  min-width: 1px;
  outline: none;
  display: flex;
  align-items: flex-start;
  justify-content: flex-start;
  z-index: 1;
  padding: inherit;
  line-height: inherit; /* 继承行高 */
  text-align: start;
}

/* 有内容时的可编辑区域 */
.has-content .editable-content {
  position: relative;
  padding: 0;
}

.editable-content:empty::before {
  content: "\200B"; /* 零宽空格，确保空字段也有高度 */
}

.preview-container {
  margin-top: 20px;
  padding: 12px;
  border-radius: 8px;
  background-color: #f5f7f9;
}

.preview-title {
  font-weight: bold;
  margin-bottom: 8px;
  color: #333;
}

.preview-content {
  line-height: 1.5;
  color: #444;
}

.submit-button {
  margin-top: 16px;
  padding: 8px 16px;
  background-color: #0078ff;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 16px;
  transition: background-color 0.2s;
}

.submit-button:hover {
  background-color: #0056cc;
}

.submit-button:disabled {
  background-color: #ccc;
  cursor: not-allowed;
}
</style>
