<template>
  <div :class="$.app">
    <div :class="$.container">
      <div
        v-if="sourceViews.generated"
        :class="$.sourceViewContainer"
        @dragover.prevent
        @drop.prevent="onDrop($event, readGenerated)"
      >
        <SourceView
          ref="generatedView"
          v-if="generatedView"
          showLineNumber
          :class="$.sourceView"
          :content="generatedView"
          @hover="onHover"
          @select="onSelectGenerated"
          @scroll.native="onScrollSync($event, 'mappingsView')"
        />
        <div :class="$.placeholderContainer" v-else>
          <input :class="$.filePicker" type="file" multiple @change="onPickFiles($event, readGenerated)" />
          <div :class="$.placeholder">
            <div :class="$.placeholderText">Drop {{missingGenerated || 'generated file'}} here</div>
          </div>
        </div>
      </div>
      <div
        v-if="sourceViews.sourceMap"
        :class="$.sourceViewContainer"
        @dragover.prevent
        @drop.prevent="onDrop($event, readSourceMap)"
      >
        <SourceView
          ref="mappingsView"
          v-if="mappingsView"
          :class="$.sourceView"
          :content="mappingsView"
          @hover="onHover"
          @select="onSelectGenerated"
          @scroll.native="onScrollSync($event, 'generatedView')"
        />
        <div :class="$.placeholderContainer" v-else>
          <input :class="$.filePicker" type="file" multiple @change="onPickFiles($event, readSourceMap)" />
          <div :class="$.placeholder">
            <div :class="$.placeholderText">Drop {{missingSourceMap || 'sourcemap file'}} here</div>
          </div>
        </div>
      </div>
      <div
        v-if="sources && selectedSource != null && sourceViews.original"
        :class="$.sourceViewContainer"
        @dragover.prevent
        @drop.prevent="onDrop($event, readOriginal)"
      >
        <SourceView
          ref="selectedView"
          v-if="selectedView"
          showLineNumber
          :class="$.sourceView"
          :content="selectedView"
          @hover="onHover"
          @select="onSelectSource"
        />
        <div :class="$.placeholderContainer" v-else>
          <input :class="$.filePicker" type="file" multiple @change="onPickFiles($event, readOriginal)" />
          <div :class="$.placeholder">
            <div :class="$.placeholderText">Drop {{path.basename(sources[selectedSource].path)}} here</div>
            <div :class="$.placeholderTips">{{sources[selectedSource].path}}</div>
          </div>
        </div>
      </div>
    </div>
    <MenuView>
      <a href="javascript:" @click="toggleView('generated')">Generated</a>
      <a href="javascript:" @click="toggleView('sourceMap')">SourceMap</a>
      <a href="javascript:" @click="toggleView('original')">Original</a>
    </MenuView>
  </div>
</template>

<script>
  import path from 'path'

  import get from 'lodash/get'
  import map from 'lodash/map'
  import find from 'lodash/find'
  import uniq from 'lodash/uniq'
  import filter from 'lodash/filter'
  import isEqual from 'lodash/isEqual'
  import sortedIndexBy from 'lodash/sortedIndexBy'

  import MenuView from './MenuView'
  import SourceView from './SourceView'

  import readAsText from '../utils/readAsText'
  import cssAttrSelector from '../utils/cssAttrSelector'
  import extractMappings from '../utils/extractMappings'
  import extractSourceMap from '../utils/extractSourceMap'

  export default {

    components: { SourceView, MenuView },

    data() {
      return {

        generatedContent: undefined,

        missingGenerated: undefined,
        missingSourceMap: undefined,

        originalMappings: undefined,
        generatedMappings: undefined,

        sources: undefined,

        highlightSource: undefined,
        selectedSource: undefined,

        sourceViews: { generated: true, sourceMap: true, original: true },
      }
    },

    created() {
      this.path = path
    },

    mounted() {
      this.highlightStyleTag = document.createElement('style')
      document.head.appendChild(this.highlightStyleTag)
    },

    beforeDestroy() {
      document.head.removeChild(this.highlightStyleTag)
    },

    methods: {

      onDrop(event, defaultHandler) {
        this.pickFiles(event.dataTransfer.files, defaultHandler)
      },

      onPickFiles(event, defaultHandler) {
        this.pickFiles(event.target.files, defaultHandler)
      },

      pickFiles(files, defaultHandler) {
        if (!files.length) {
          return
        }
        if (files.length === 1) {
          defaultHandler.call(this, files[0])
          return
        }
        if (files[0].name.endsWith('.map') || files[0].name.endsWith('.json')) {
          this.readSourceMap(files[0])
          this.readGenerated(files[1])
        } else {
          this.readSourceMap(files[1])
          this.readGenerated(files[0])
        }
      },

      async readGenerated(file) {
        this.generatedContent = await readAsText(file)
        const sourceMap = extractSourceMap(this.generatedContent)
        if (this.generatedMappings && sourceMap.sourceMapFile) {
          return
        }
        this.setSourceMap(sourceMap)
        this.selectedSource = this.sources && 0
      },

      async readSourceMap(file) {
        this.setSourceMap(extractMappings(JSON.parse(await readAsText(file))))
        this.selectedSource = Math.min(this.sources.length - 1, this.selectedSource || 0)
      },

      async readOriginal(file) {
        const { selectedSource } = this
        if (selectedSource != null) {
          this.sources[selectedSource].content = await readAsText(file)
        }
      },

      setSourceMap(sourceMap) {
        this.missingGenerated = sourceMap.generatedFile
        this.missingSourceMap = sourceMap.sourceMapFile

        this.sources = sourceMap.sources

        this.originalMappings = Object.freeze(sourceMap.originalMappings)
        this.generatedMappings = Object.freeze(sourceMap.generatedMappings)
      },

      onHover(generatedPosition) {
        if (!isEqual(this.highlightSource, generatedPosition)) {
          this.highlightSource = Object.freeze(generatedPosition)
        }
      },

      onSelectGenerated(chunk) {
        this.selectedSource = chunk.source
        this.$nextTick(() => {
          this.scrollToLine('selectedView', chunk.line)
        })
      },

      onSelectSource(chunk) {
        const targetMappings = filter(this.generatedMappings, {
          source: chunk.source,
          originalLine: chunk.line,
          originalColumn: chunk.column,
        })
        this.scrollToLine('mappingsView', uniq(map(targetMappings, 'generatedLine'))[0])
      },

      scrollToLine(target, line) {
        const view = this.$refs[target]
        if (view) {
          view.scrollToLine(line)
        }
      },

      onScrollSync(event, target) {
        if (!this.$refs[target]) {
          return
        }
        const { $el } = this.$refs[target]
        if (!$el || event.target.scrollTop === this.lastScrollTopSync) {
          return
        }
        $el.scrollTop = event.target.scrollTop
        this.lastScrollTopSync = $el.scrollTop
      },

      toggleView(type) {
        this.sourceViews[type] = !this.sourceViews[type]
      },
    },

    computed: {

      generatedView() {

        if (!this.generatedContent) {
          return
        }

        const rawLines = this.generatedContent.split(/\r?\n/g)
        const lines = rawLines.map(() => [])

        const generatedMappings = this.generatedMappings || []

        for (let i = generatedMappings.length - 1; i >= 0; i--) {
          const mapping = generatedMappings[i]
          const lineNumber = mapping.generatedLine - 1

          if (!lines[lineNumber]) {
            continue
          }

          lines[lineNumber].unshift({
            names: [mapping.name],
            source: mapping.source,
            line: mapping.originalLine,
            column: mapping.originalColumn,
            content: rawLines[lineNumber].slice(mapping.generatedColumn),
          })

          rawLines[lineNumber] = rawLines[lineNumber].slice(0, mapping.generatedColumn)
        }

        for (let i = 0, ii = rawLines.length; i < ii; i++) {
          if (rawLines[i]) {
            lines[i].unshift({ names: [], content: rawLines[i] })
          }
        }

        return lines
      },

      mappingsView() {

        if (!this.generatedMappings) {
          return
        }

        const lines = []

        if (this.generatedContent) {
          lines.length = this.generatedContent.split(/\r?\n/g).length
        }

        for (const mapping of this.generatedMappings) {
          const lineNumber = mapping.generatedLine - 1

          if (!lines[lineNumber]) {
            lines[lineNumber] = []
          }

          let source = this.sources[mapping.source]

          if (source) {
            source = source.path
          }

          lines[lineNumber].push({
            names: [source, mapping.name],
            source: mapping.source,
            line: mapping.originalLine,
            column: mapping.originalColumn,
            content: `${mapping.generatedColumn} => ${mapping.originalLine}:${mapping.originalColumn}`,
          })

          lines[lineNumber].push({ names: [], content: ' ' })
        }

        return lines
      },

      selectedView() {
        const source = this.sources[this.selectedSource]

        if (!source.content) {
          return
        }

        const rawLines = source.content.split(/\r?\n/g)
        const lines = rawLines.map(() => [])

        const beginIndex = sortedIndexBy(this.originalMappings || [], this.selectedSource, 'source')

        for (let i = beginIndex - 1; i >= 0; i--) {
          const mapping = this.originalMappings[i]

          if (mapping.source !== this.selectedSource) {
            continue
          }

          const lineNumber = mapping.originalLine - 1

          if (!lines[lineNumber]) {
            continue
          }

          if (get(lines[lineNumber][0], 'column') === mapping.originalColumn) {
            lines[lineNumber][0].names.push(mapping.name)
            continue
          }

          lines[lineNumber].unshift({
            names: [mapping.name],
            source: mapping.source,
            line: mapping.originalLine,
            column: mapping.originalColumn,
            content: rawLines[lineNumber].slice(mapping.originalColumn),
          })

          rawLines[lineNumber] = rawLines[lineNumber].slice(0, mapping.originalColumn)
        }

        for (let i = 0, ii = rawLines.length; i < ii; i++) {
          if (rawLines[i]) {
            lines[i].unshift({ names: [], content: rawLines[i] })
          }
        }

        return lines
      },
    },

    watch: {

      highlightSource(newSource) {
        const chunkSelector = cssAttrSelector({
          'data-origin-source': newSource.source,
          'data-origin-line': newSource.line,
          'data-origin-column': newSource.column,
        })
        this.highlightStyleTag.innerHTML = `
          ${chunkSelector} {
            color: white;
            background: black;
            border-left-color: black !important;
          }
        `
      },
    },
  }
</script>

<style>
  html,
  body {
    height: 100%;
    margin: 0;
  }
  *,
  *:before,
  *:after {
    box-sizing: border-box;
  }
</style>

<style module="$">
  .app {
    position: relative;
    height: 100%;
    width: 100%;
  }
  .container {
    display: flex;
    width: 100%;
    height: 100%;
  }
  .container > * {
    flex: 1 auto;
    width: 1px;
  }
  .placeholderContainer {
    position: relative;
    overflow: hidden;
    height: 100%;
    padding: 5px;
  }
  .filePicker {
    position: absolute;
    top: 0;
    right: 0;
    width: 100%;
    height: 100%;
    opacity: 0;
    cursor: pointer;
  }
  .placeholder {
    display: flex;
    flex-direction: column;
    justify-content: center;
    text-align: center;
    height: 100%;
    font-family: sans-serif;
    border: 2px dashed gray;
  }
  .placeholderText {
    display: inline-block;
    width: 100%;
    overflow: hidden;
    text-overflow: ellipsis;
    font-size: 1.5em;
    color: gray;
  }
  .placeholderTips {
    white-space: nowrap;
    font-size: .9em;
    overflow: hidden;
    text-overflow: ellipsis;
    color: #999;
  }
  .sourceView {
    height: 100%;
  }
</style>
