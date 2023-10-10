<script setup lang="ts">
import { fromSource } from 'convert-source-map'
import { getHighlighter } from 'shikiji'
import githubLightTheme from '~/constants/github-light-theme'

const sourceMapRE = /\/\/#\s*sourceMappingURL=.*\n?/g

const generatedCode = useLocalStorage('source-map-visualization-generated-code', '')
const sourcemapCode = useLocalStorage('source-map-visualization-sourcemap', '')
watch(generatedCode, (code) => {
  if (sourceMapRE.test(code))
    sourcemapCode.value = code.match(sourceMapRE)?.[0] || ''
})

const monacoRef = shallowRef<HTMLDivElement>()
if (process.client) {
  const monaco = await import('monaco-editor/esm/vs/editor/editor.api')
  const { editor, languages } = monaco
  languages.register({
    id: 'typescript',
  })
  editor.defineTheme('github-light', githubLightTheme)
  editor.setTheme('github-light')
  const model = editor.createModel(generatedCode.value, 'typescript')
  watch(monacoRef, (el) => {
    if (el) {
      const editorInstance = editor.create(el, {
        value: generatedCode.value,
        selectOnLineNumbers: true,
        automaticLayout: true,
        minimap: {
          enabled: false,
        },
        language: 'typescript',
        model,
      })
      editorInstance.render()
    }
  })
  model.onDidChangeContent((code) => {
    generatedCode.value = code.changes[0].text
  })
}
const sourcemap = computed(() => {
  return fromSource(sourcemapCode.value)?.sourcemap
})

const shiki = await getHighlighter({
  themes: ['github-light'],
  langs: ['javascript', 'typescript'],
})

function codeToHtml(filename: string, code: string) {
  return shiki.codeToHtml(code, { lang: filename.endsWith('.ts') ? 'typescript' : 'javascript', theme: 'github-light' })
}
</script>

<template>
  <div h-screen w-screen>
    <Suspense>
      <ClientOnly>
        <div border="2px" h-full w-full grid="~ cols-2">
          <div h-full w-full>
            <div ref="monacoRef" h-full w-full />
          </div>

          <div h-full w-full>
            <div v-for="i in sourcemap?.sources.length" :key="i" flex="~ col" border="1px" w-full overflow-hidden p-2>
              <p border-b-1 text-blue-600>
                {{ sourcemap.sources[i - 1] }}
              </p>
              <div w-full overflow-x-scroll py-2 v-html="codeToHtml(sourcemap.sources[i - 1], sourcemap.sourcesContent[i - 1])" />
            </div>
          </div>
        </div>
      </ClientOnly>

      <template #fallback>
        <div italic op50>
          <span animate-pulse>Loading...</span>
        </div>
      </template>
    </Suspense>
  </div>
</template>
