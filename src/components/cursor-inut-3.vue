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



// 更新字段值
const updateFieldValue = (index, event) => {
  templateParts.value[index].value = event.target.textContent;
  showPreview.value = isTemplateComplete.value;
};

// 失去焦点时的处理
const onFieldBlur = (index) => {
  // 如果需要在失焦时执行额外逻辑，可以在这里添加
};

// 增强onKeyDown函数，确保可编辑元素可以正确导航
const onKeyDown = (event, index) => {
  console.log('编辑区域键盘事件:', event.key, index);
  
  // 获取可编辑元素
  const target = event.target;
  const textContent = target.textContent || '';
  
  // 获取当前光标位置，考虑空内容情况
  const selection = window.getSelection();
  const isAtStart = selection.anchorOffset === 0;
  const isAtEnd = selection.anchorOffset >= textContent.length;

  console.log('编辑区域光标情况:', '内容:', JSON.stringify(textContent),
             '是否在开头:', isAtStart, '是否在末尾:', isAtEnd,
             '光标位置:', selection.anchorOffset);
  
  // 新增：处理 Backspace 键在光标位于内容开头时的情况
  if (event.key === 'Backspace' && isAtStart) {
    console.log('可编辑区域开头按下Backspace键');
    event.preventDefault();
    event.stopPropagation();
    // 将光标移动到前一个元素
    navigateToAdjacentElement(index, -1);
    return false;
  }
  
  // 键盘导航逻辑
  if (event.key === 'ArrowLeft' && isAtStart) {
    // 向左导航
    console.log('从可编辑区域向左导航');
    event.preventDefault();
    event.stopPropagation();
    navigateToAdjacentElement(index, -1);
    return false;
  }
  else if (event.key === 'ArrowRight' && isAtEnd) {
    // 向右导航
    console.log('从可编辑区域向右导航');
    event.preventDefault();
    event.stopPropagation();
    navigateToAdjacentElement(index, 1);
    return false;
  }
  else if (event.key === 'Tab') {
    // Tab 键导航
    event.preventDefault();
    event.stopPropagation();
    if (event.shiftKey) {
      navigateToAdjacentElement(index, -1);
    } else {
      navigateToAdjacentElement(index, 1);
    }
    return false;
  }
  
  // 不阻止其他按键的默认行为，允许正常编辑
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

const handleGlobalKeydown = (event) => {
  // 获取当前活动元素
  const activeElement = document.activeElement;
  
  // 判断活动元素是否是文本内容元素（不处理editable-content）
  const isTextContent = activeElement.classList.contains('text-content');
  
  // 如果不是文本内容元素，则不处理
  if (!isTextContent) return;
  
  // 获取当前元素的索引
  const partIndex = parseInt(activeElement.dataset.partIndex);
  if (isNaN(partIndex)) {
    console.error('无法获取有效的partIndex');
    return;
  }
  
  // 检查光标位置
  const selection = window.getSelection();
  const isAtStart = selection.anchorOffset === 0;
  
  // 改进end位置检测，增加记录日志
  const textLength = activeElement.textContent.length;
  const cursorPos = selection.anchorOffset;
  const isAtEnd = activeElement.textContent && cursorPos === textLength;
  
  console.log('文本区域光标情况:', '索引:', partIndex, '文本长度:', textLength, 
              '光标位置:', cursorPos, '是否在末尾:', isAtEnd);
  
  // 新增：处理Backspace键在文本区域开头的情况
  if (event.key === 'Backspace' && isAtStart && partIndex > 0) {
    const prevPart = templateParts.value[partIndex - 1];
    // 如果前一个元素是占位符，删除它
    if (prevPart.type === 'placeholder') {
      event.preventDefault();
      deletePlaceholder(partIndex - 1);
      return;
    }
  }
              
  // 只处理文本区域的键盘导航
  if (event.key === 'ArrowLeft' && isAtStart) {
    console.log('从文本区域向左导航');
    navigateToAdjacentElement(partIndex, -1);
    event.preventDefault();
  }
  else if (event.key === 'ArrowRight' && isAtEnd) {
    console.log('从文本区域向右导航到索引:', partIndex + 1);
    navigateToAdjacentElement(partIndex, 1);
    event.preventDefault();
  }
  else if (event.key === 'Tab') {
    event.preventDefault();
    if (event.shiftKey) {
      navigateToAdjacentElement(partIndex, -1);
    } else {
      navigateToAdjacentElement(partIndex, 1);
    }
  }
};

// 改进focusEditableField函数，优化光标设置逻辑
const focusEditableField = async (index, position = 'start') => {
  await nextTick();
  
  console.log('focusEditableField被调用，索引:', index, '位置:', position);
  
  // 获取所有可编辑字段元素
  const fields = document.querySelectorAll('.editable-content');
  
  // 找到目标字段
  let targetField = null;
  for (const field of fields) {
    const fieldPartIndex = parseInt(field.dataset.partIndex);
    if (fieldPartIndex === index) {
      targetField = field;
      console.log('找到目标字段，索引:', fieldPartIndex);
      break;
    }
  }
  
  if (targetField) {
    console.log('准备聚焦目标字段,HTML内容:', targetField.innerHTML);
    
    // 1. 先确保字段可见
    targetField.style.display = 'block';
    
    // 2. 聚焦目标字段
    targetField.focus();
    
    // 3. 设置光标位置
    const range = document.createRange();
    const sel = window.getSelection();
    
    try {
      // 更加精确地定位光标位置
      if (position === 'end') {
        // 在末尾
        if (targetField.lastChild && targetField.lastChild.nodeType === Node.TEXT_NODE) {
          range.setStart(targetField.lastChild, targetField.lastChild.length);
        } else if (targetField.lastChild) {
          range.setStartAfter(targetField.lastChild);
        } else {
          range.setStart(targetField, 0);
        }
      } else {
        // 在开头
        if (targetField.firstChild && targetField.firstChild.nodeType === Node.TEXT_NODE) {
          range.setStart(targetField.firstChild, 0);
        } else if (targetField.firstChild) {
          range.setStartBefore(targetField.firstChild);
        } else {
          range.setStart(targetField, 0);
        }
      }
      
      range.collapse(true);
      sel.removeAllRanges();
      sel.addRange(range);
      
      // 4. 确认光标位置已设置
      console.log('光标已设置到:', position, 
                  '位置:', sel.anchorOffset, 
                  '字段内容长度:', targetField.textContent.length);
                  
    } catch (e) {
      console.error('设置光标位置失败:', e);
    }
  } else {
    console.error('未找到索引为', index, '的可编辑字段');
  }
};

// 添加缺失的focusTextElement函数
const focusTextElement = async (index, position = 'start') => {
  await nextTick();
  
  console.log('focusTextElement被调用，索引:', index, '位置:', position);
  
  // 获取所有文本内容元素
  const textElements = document.querySelectorAll('.text-content');
  
  // 找到目标文本元素
  let targetElement = null;
  for (const element of textElements) {
    const elementIndex = parseInt(element.dataset.partIndex);
    if (elementIndex === index) {
      targetElement = element;
      console.log('找到目标文本元素，索引:', elementIndex);
      break;
    }
  }
  
  if (targetElement) {
    // 聚焦目标元素
    targetElement.focus();
    
    // 设置光标位置
    const range = document.createRange();
    const sel = window.getSelection();
    
    try {
      if (position === 'end') {
        if (targetElement.lastChild) {
          range.setStartAfter(targetElement.lastChild);
        } else {
          range.setStart(targetElement, targetElement.textContent.length);
        }
      } else {
        if (targetElement.firstChild) {
          range.setStartBefore(targetElement.firstChild);
        } else {
          range.setStart(targetElement, 0);
        }
      }
      
      range.collapse(true);
      sel.removeAllRanges();
      sel.addRange(range);
    } catch (e) {
      console.error('设置文本光标位置失败:', e);
    }
  } else {
    console.error('未找到索引为', index, '的文本元素');
  }
};

// 修改navigateToAdjacentElement函数，添加延时确保聚焦成功
const navigateToAdjacentElement = (currentIndex, direction) => {
  const targetIndex = currentIndex + direction;
  
  console.log('导航到相邻元素:', '当前索引:', currentIndex, '方向:', direction, '目标索引:', targetIndex);
  
  // 检查索引是否在范围内
  if (targetIndex < 0 || targetIndex >= templateParts.value.length) {
    console.log('目标索引超出范围');
    return; // 已经到达边界，不执行任何操作
  }
  
  const targetPart = templateParts.value[targetIndex];
  console.log('目标元素类型:', targetPart.type);
  
  // 添加延时，确保DOM处理完成
  setTimeout(() => {
    if (targetPart.type === 'text') {
      // 如果是普通文本，聚焦文本元素
      focusTextElement(targetIndex, direction < 0 ? 'end' : 'start');
    } else if (targetPart.type === 'placeholder') {
      // 如果是占位符，聚焦可编辑字段
      focusPlaceholder(targetIndex, direction < 0 ? 'end' : 'start');
    }
  }, 0);
};

// 更新focusPlaceholder函数
const focusPlaceholder = (index, position = 'start') => {
  focusEditableField(index, position);
};

const deletePlaceholder = async (index) => {
  console.log('删除占位符，索引:', index);
  
  // 确保要删除的是占位符
  if (templateParts.value[index].type !== 'placeholder') {
    console.error('尝试删除非占位符元素');
    return;
  }
  
  // 找到删除后需要聚焦的元素索引（通常是前一个元素）
  let focusIndex = index - 1;
  if (focusIndex < 0) {
    // 如果没有前一个元素，则聚焦后一个元素
    focusIndex = index + 1;
    if (focusIndex >= templateParts.value.length) {
      // 如果也没有后一个元素，则不执行删除
      return;
    }
  }
  
  // 执行删除操作
  templateParts.value.splice(index, 1);
  
  // 更新引用数组
  editableFields.value = [];
  
  // 等待DOM更新
  await nextTick();
  
  // 聚焦到相应元素
  const focusTarget = templateParts.value[focusIndex];
  if (focusTarget.type === 'text') {
    focusTextElement(focusIndex, 'end');
  } else {
    focusPlaceholder(focusIndex, 'end');
  }
  
  // 更新预览状态
  showPreview.value = isTemplateComplete.value;
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
  color: #0057ff66;
  font-weight: 600;
  opacity: 0.8;
  pointer-events: none;
  white-space: nowrap;
}

/* 可编辑内容 */
.editable-content {
  position: absolute;
  top: 0;
  right: 0;
  bottom: 0;
  left: 0;
  min-width: 1px;
  outline: none;
  display: block; /* 修改为block以确保元素始终可见 */
  z-index: 1;
  padding: inherit;
  line-height: inherit;
  text-align: start;
  color: #0057ff;
  font-weight: 600;

  caret-color: #0078ff; /* 增加光标颜色以增强可见性 */
}

/* 有内容时的可编辑区域 */
.has-content .editable-content {
  position: relative; /* 保持相对定位 */
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
