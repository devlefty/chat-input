<template>
  <div class="chat-input-container">
    <span class="text-default" contenteditable="true">
      我是一个 
    </span>
    <span
      ref="editableArea"
      class="text-input"
      contenteditable="true"
      @input="onInput"
      @paste="handlePaste"
      @focus="onFocus"
      @blur="onBlur"
      placeholder="[输入职业]"
    ></span>
    <span class="text-default" contenteditable="true">
      你好啊
    </span>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue';

// 响应式数据
const message = ref('');
const editableArea = ref(null);

// 处理输入事件
const onInput = (e) => {
  message.value = e.target.innerHTML;
  
  // 根据是否有内容添加或删除focused类
  if (e.target.innerHTML !== '') {
    e.target.classList.add('focused');
  } else {
    e.target.classList.remove('focused');
  }
};

// 处理粘贴，只保留纯文本
const handlePaste = (e) => {
  e.preventDefault();
  const text = e.clipboardData?.getData('text/plain') || '';
  document.execCommand('insertText', false, text);
};

// 点击容器时聚焦到输入区域
const focusInput = () => {
  editableArea.value?.focus();
};

// 修改处理聚焦事件
function onFocus() {
  const el = editableArea.value;
  
  // 只有当有内容时才添加focused类
  if (el.innerHTML !== '') {
    el.classList.add('focused');
  }
  
  // 如果内容为空，确保光标在最左侧
  if (el.innerHTML === '') {
    const range = document.createRange();
    const selection = window.getSelection();
    range.setStart(el, 0);
    range.collapse(true);
    selection.removeAllRanges();
    selection.addRange(range);
  }
}

// 处理失焦事件
function onBlur() {
  const el = editableArea.value;
  if (el.innerHTML === '') {
    el.classList.remove('focused');
  }
}

// 在组件挂载后设置初始状态
onMounted(() => {
  const el = editableArea.value;
  
  if (el && el.innerHTML === '') {
    // 确保初始状态下占位符可见
    el.classList.remove('focused');
  }
});
</script>

<style scoped>
.chat-input-container {
  display: flex;
  padding: 12px;
  border: 1px solid #ddd;
  border-radius: 16px;
  overflow: hidden;
  background-color: #fff;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.1);
  margin: 10px;
  line-height: 32px;
  cursor: text; /* 添加文本光标，提示用户这是可以输入的区域 */
  align-items: center;
}

.text-default {
  white-space: nowrap;
  outline: none;
  cursor: text;
  pointer-events: none;
}

.text-input {
  width: fit-content;
  min-width: fit-content;
  min-height: 24px;
  outline: none;
  font-size: 14px;
  word-break: break-word;
  text-align: left;
  white-space: pre-wrap;
  border-radius: 10px;
  color: #0057ff99;
  pointer-events: none;
  text-align: left;
  background: #0057ff0f;
  padding: 1px 8px;
  font-weight: 600;
  display: inline-block;
  border-radius: 10px;
  word-break: break-word;
}

/* 修改Placeholder效果，简化选择器 */
.text-input:empty:before {
  content: attr(placeholder);
  cursor: text;
}

/* 当focused类存在时隐藏占位符 */
.text-input.focused:before {
  content: '';
}
</style>