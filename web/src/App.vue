<template>
  <div class="flex flex-col h-screen bg-gray-900 p-4 text-gray-100">
    <!-- 顶部栏 -->
    <div class="flex justify-between items-center mb-4 flex-shrink-0">
      <h1 class="text-2xl font-bold">示例题目：两数之和</h1>
      <div class="flex space-x-2">
        <select v-model="language" class="bg-gray-800 border-gray-700 text-gray-100 rounded px-2 py-1">
          <option v-for="(val, key) in languageMap" :key="key" :value="key">
            {{ key.toUpperCase() }}
          </option>
        </select>
        
        <button @click="runCode" class="bg-blue-700 hover:bg-blue-600 px-4 py-2 rounded" :disabled="!!testCaseError">
          运行代码
        </button>
        <button @click="submitCode" class="bg-green-700 hover:bg-green-600 px-4 py-2 rounded">
          提交
        </button>
      </div>
    </div>

    <!-- 主体布局：编辑器 + 右侧信息栏-->
    <div class="flex flex-1 gap-0 relative overflow-hidden" :style="{ height: `calc(100% - ${outputHeight + 8}px)` }">
      <!-- 左侧编辑器 -->
      <div class="border rounded overflow-hidden bg-gray-800 flex flex-col" :style="{ width: `${editorWidth}%` }">
        <MonacoEditor 
          v-model="code" 
          :language="languageMap[language]" 
          height="100%"
          theme="vs-dark"
          @change="val => code = val" 
          :options="editorOptions" 
        />
      </div>

      <!-- 垂直拖拽分隔条 -->
      <div 
        class="w-1 bg-gray-700 hover:bg-blue-500 cursor-col-resize transition-colors relative group flex-shrink-0"
        @mousedown="startResizeHorizontal"
      >
        <div class="absolute inset-y-0 -left-1 -right-1"></div>
        <div class="absolute top-1/2 left-1/2 transform -translate-x-1/2 -translate-y-1/2 opacity-0 group-hover:opacity-100 bg-blue-500 px-1 py-2 rounded text-xs pointer-events-none">
          ⋮
        </div>
      </div>

      <!-- 右侧信息栏 -->
      <div 
        class="bg-gray-800 p-4 rounded shadow overflow-y-auto flex-shrink-0"
        :style="{ width: `calc(${100 - editorWidth}% - 4px)` }"
      >
        <h2 class="font-semibold mb-2 text-gray-100">题目描述</h2>
        <p>test-test-test</p>

        <h3 class="font-semibold mt-4 text-gray-100">示例输入输出</h3>
        <pre class="bg-gray-700 p-2 rounded text-gray-100">
输入: nums = [2,7], target = 9
输出: 9
解释: 因为 nums[0] + nums[1] == 9
        </pre>

        <h3 class="font-semibold mt-4 text-gray-100">测试用例</h3>
        <ul v-if="testCases.length > 0">
          <li v-for="(t, i) in testCases" :key="i" class="mb-1">
            {{ i + 1 }}. 输入: {{ t.input }} → 期望: {{ t.expected }}
          </li>
        </ul>
        <p v-else class="text-gray-400">暂无测试用例</p>
      </div>
    </div>

    <!-- 水平拖拽分隔条 -->
    <div 
      class="h-1 bg-gray-700 hover:bg-blue-500 cursor-row-resize transition-colors my-2 relative group flex-shrink-0"
      @mousedown="startResizeVertical"
    >
      <div class="absolute inset-x-0 -top-1 -bottom-1"></div>
      <div class="absolute top-1/2 left-1/2 transform -translate-x-1/2 -translate-y-1/2 opacity-0 group-hover:opacity-100 bg-blue-500 px-2 py-1 rounded text-xs pointer-events-none">
        ⋯
      </div>
    </div>

    <!-- 输出面板 -->
    <div 
      class="p-2 border rounded bg-gray-800 overflow-y-auto relative text-gray-100 flex-shrink-0"
      :style="{ height: `${outputHeight}px` }"
    >
      <h3 class="font-semibold mb-2">运行结果</h3>
      <pre class="whitespace-pre-wrap">{{ output || '等待运行...' }}</pre>
      <button 
        @click="copyOutput" 
        class="absolute top-2 right-2 bg-gray-700 hover:bg-gray-600 px-2 py-1 rounded"
        :disabled="!output"
      >
        复制
      </button>
    </div>
  </div>
</template>

<script setup>
import { ref, watch, onMounted, onUnmounted } from 'vue'
import MonacoEditor from 'monaco-editor-vue3'
import axios from 'axios'

// 语言映射
const languageMap = { cpp: 'cpp', python: 'python', c: 'c' }

const language = ref('python')
const code = ref("")
const testCasesInput = ref('[{"input":[2,7],"expected":9},{"input":[3,2],"expected":5}]')
const output = ref('')
const testCaseError = ref('')
const testCases = ref([])

// 窗口大小控制
const editorWidth = ref(65) // 编辑器宽度百分比
const outputHeight = ref(160) // 输出面板高度(px)

// 拖拽状态
const isResizingHorizontal = ref(false)
const isResizingVertical = ref(false)

// Monaco 编辑器配置
const editorOptions = {
  fontSize: 14,
  fontFamily: 'Fira Code, Monaco, Consolas',
  tabSize: 4,
  insertSpaces: true,
  automaticLayout: true,
  suggest: {
    enabled: true,
    maxVisibleSuggestions: 20,
    showWords: true,
    showSnippets: true,
    showIssues: true,
  },
  quickSuggestions: {
    other: true,
    comments: false,
    strings: false,
  },
  quickSuggestionsDelay: 300,
  parameterHints: {
    enabled: true,
    cycle: true,
  },
  autoClosingBrackets: 'always',
  autoClosingQuotes: 'always',
  autoClosingOvertype: 'always',
  formatOnPaste: true,
  formatOnType: true,
  wordBasedSuggestions: 'matchingDocuments',
  acceptSuggestionOnCommitCharacter: true,
  acceptSuggestionOnEnter: 'on',
  minimap: { enabled: true },
  scrollBeyondLastLine: false,
  renderWhitespace: 'none',
  cursorBlinking: 'blink',
  cursorSmoothCaretAnimation: 'on',
}

// 水平调整大小（左右分栏）
const startResizeHorizontal = (e) => {
  isResizingHorizontal.value = true
  document.body.style.cursor = 'col-resize'
  document.body.style.userSelect = 'none'
  e.preventDefault()
}

// 垂直调整大小（上下分栏）
const startResizeVertical = (e) => {
  isResizingVertical.value = true
  document.body.style.cursor = 'row-resize'
  document.body.style.userSelect = 'none'
  e.preventDefault()
}

const handleResizeVertical = (e) => {
  if (!isResizingVertical.value) return
  
  const windowHeight = window.innerHeight
  const newHeight = windowHeight - e.clientY - 16 // 减去padding
  
  // 限制范围 100px - 500px
  outputHeight.value = Math.min(Math.max(newHeight, 100), 500)
}

const stopResize = () => {
  isResizingHorizontal.value = false
  isResizingVertical.value = false
  document.body.style.cursor = 'default'
  document.body.style.userSelect = 'auto'
}

// 全局鼠标事件监听
const handleMouseMove = (e) => {
  if (isResizingHorizontal.value) {
    const container = document.querySelector('.flex.flex-1.gap-0')
    if (container) {
      const rect = container.getBoundingClientRect()
      const offsetX = e.clientX - rect.left
      const newWidth = (offsetX / rect.width) * 100
      editorWidth.value = Math.min(Math.max(newWidth, 30), 80)
    }
  }
  
  if (isResizingVertical.value) {
    handleResizeVertical(e)
  }
}

const handleMouseUp = () => {
  stopResize()
}

onMounted(() => {
  document.addEventListener('mousemove', handleMouseMove)
  document.addEventListener('mouseup', handleMouseUp)
})

onUnmounted(() => {
  document.removeEventListener('mousemove', handleMouseMove)
  document.removeEventListener('mouseup', handleMouseUp)
})

// 测试用例解析
watch(testCasesInput, val => {
  try {
    const parsed = JSON.parse(val)
    if (!Array.isArray(parsed)) throw new Error('必须是数组')
    testCases.value = parsed
    testCaseError.value = ''
  } catch (e) {
    testCaseError.value = '❌ 测试用例必须是有效的 JSON 数组'
    testCases.value = []
  }
}, { immediate: true })

const extractFunctionName = (codeStr, lang) => {
  let cleanCode = codeStr
    .replace(/\/\/.*$/gm, '')
    .replace(/\/\*[\s\S]*?\*\//g, '')
    .replace(/^\s*#include\s+.*$/gm, '')
    .replace(/^\s*#define\s+.*$/gm, '')
    .replace(/^\s*#pragma\s+.*$/gm, '')

  if (lang === 'python') {
    const m = cleanCode.match(/^\s*def\s+(\w+)\s*\(/m)
    return m ? m[1] : ''
  }

  if (lang === 'cpp') {
    const m = cleanCode.match(/(?:int|void|bool|char|float|double|long|size_t|auto|string|vector\s*<[^>]+>|std::\w+(?:\s*<[^>]+>)?|\w+\s*\*+)\s+(\w+)\s*\(/)
    return m ? m[1] : ''
  }

  if (lang === 'c') {
    const m = cleanCode.match(/(?:int|void|char|float|double|long|size_t|unsigned\s+(?:int|long|char)|\w+\s*\*+)\s+(\w+)\s*\(/)
    return m ? m[1] : ''
  }

  return ''
}

const runCode = async () => {
  if (testCaseError.value) {
    output.value = testCaseError.value
    return
  }

  if (testCases.value.length === 0) {
    output.value = '❌ 没有测试用例'
    return
  }

  const funcName = extractFunctionName(code.value, language.value)
  if (!funcName) {
    output.value = '❌ 无法识别函数名，请检查代码格式'
    return
  }

  output.value = '⏳ 正在执行...'

  try {
    const resp = await axios.post('http://localhost:3000/run', {
      language: language.value,
      code: code.value,
      function: funcName,
      test_cases: testCases.value
    }, { timeout: 10000 })

    const data = resp.data
    let out = `语言: ${data.language}\n总耗时: ${data.execution_time_ms} ms\n\n`

    if (data.output.error) {
      out += `❌ 执行错误: ${data.output.error}\n`
      output.value = out
      return
    }

    if (data.output.summary) {
      out += `总测试数: ${data.output.summary.total}\n`
      out += `通过数: ${data.output.summary.passed}\n`
      out += `通过率: ${(data.output.summary.pass_rate * 100).toFixed(1)}%\n`
      out += `时间复杂度: ${data.output.summary.time_complexity || '未知'}\n`
      out += `空间复杂度: ${data.output.summary.space_complexity || '未知'}\n\n`
    }

    if (data.output.cases && data.output.cases.length) {
      out += '测试用例详情:\n'
      data.output.cases.forEach((c, i) => {
        out += `#${i + 1} 输入: ${JSON.stringify(c.input)} | 输出: ${c.output} | 期望: ${c.expected} | ${c.passed ? '✅' : '❌'} | 时间: ${c.time_ms}ms\n`
      })
    }

    output.value = out
  } catch (err) {
    console.error(err)
    if (err.code === 'ECONNREFUSED') {
      output.value = '❌ 无法连接到后端服务器 (localhost:3000)，请确保后端已启动'
    } else if (err.response?.data?.error) {
      output.value = `❌ 执行失败: ${err.response.data.error}`
    } else if (err.message === 'timeout of 10000ms exceeded') {
      output.value = '❌ 执行超时，代码可能陷入死循环'
    } else {
      output.value = `❌ 执行失败: ${err.message}`
    }
  }
}

const submitCode = () => {
  if (testCaseError.value) {
    alert('❌ 请先修复测试用例错误')
    return
  }
  alert('✅ 提交成功！（模拟消息）')
}

const copyOutput = () => {
  if (output.value) {
    navigator.clipboard.writeText(output.value).then(() => {
      alert('✅ 已复制到剪贴板')
    }).catch(() => {
      alert('❌ 复制失败')
    })
  }
}
</script>