<template>
    <div class="data-grid-container" ref="container">
        <div class="data-grid-header clearfix">
            <h1 class="left" v-if="title">{{title}}</h1>
            <button class="data-grid-options-toggle right" @click.stop="toggleOptions">options</button>
        </div>
        <section class="data-grid-options" v-if="showOptions">
            <div>
                <input type="checkbox" v-model="stickyHeader"> <label>Sticky header</label>
            </div>
            <div v-for="(header, index) in columnOrder" :key="header">
                <label class="bold">{{header}}</label><br>
                <label>Alias</label> <input type="text" @change="setAlias(header, $event.srcElement.value)" /><br>
                <label>Visible</label> <input type="checkbox" @change="toggleHiddenColumn(header)" :checked="!hiddenColumns.includes(header)" /><br>
                <button v-if="index > 0" @click.stop="moveHeader(header, index, index - 1)">up</button>
                <button v-if="index < columnOrder.length - 1" @click.stop="moveHeader(header, index, index + 1)">down</button>
            </div>
        </section>
        <div class="data-grid-table-container">
            <section class="data-grid" :class="{sticky: stickyHeader}">
                <header class="data-grid-column-header">
                    <div v-for="(header, index) in headers" :key="header" :ref="'header-' + index">
                        <span class="header-text text-input" :title="searchPhrases[header]" @click.stop="startSearch(header)" v-if="searchField !== header">{{header}}</span>
                        <input type="text" v-model="searchInput" v-if="searchField === header" @keydown.enter.stop="stopSearch" @blur="stopSearch" ref="searchInput" >
                        <span class="sort-indicator sort-indicator-none" @click="orderBy(header)" v-if="!sortIndicators[index].isSorting"></span>
                        <span class="sort-indicator sort-indicator-up" @click="orderBy(header)" v-if="sortIndicators[index].isSorting && sortIndicators[index].direction == 1"></span>
                        <span class="sort-indicator sort-indicator-down" @click="orderBy(header)" v-if="sortIndicators[index].isSorting && sortIndicators[index].direction == -1"></span>
                    </div>
                </header>
                <div class="data-grid-row margin-top" v-if="stickyHeader">
                    <div v-for="(header, index) in filteredHeaders" :key="header" :ref="'data-' + index">
                        <span>&nbsp;</span>
                    </div>
                </div>
                <div class="data-grid-row" v-for="row in orderedRows" :key="row[rowId]" @click="rowSelected(row)">
                    <div v-for="header in filteredHeaders" :key="header">
                        <span>{{row[header]}}</span>
                    </div>
                </div>
            </section>
        </div>
        <section class="data-grid-footer">
            <p class="right">total: {{orderedRows.length}}</p>
        </section>
    </div>
</template>

<script>
export default {
  name: 'DataGrid',
  props: {
    title: Array,
    gridData: Array,
    maxRows: Number
  },
  emits: ['selected'],
  data() {
      return {
          rowId: '_id',
          sortProperty: null,
          sortDirection: 1,
          searchField: null,
          searchInput: '',
          searchPhrases: {},
          showOptions: false,
          hiddenColumns: [],
          aliases: {},
          columnOrder: [],
          stickyHeader: false
      }
  },
  mounted() {
      this.loadSettings()
      this.resized()
      new ResizeObserver(this.resized).observe(this.$refs.container)
  },
  computed: {
      rawHeaders() {
          const rowIdId = this.rowId
          return !this.gridData?.length ? [] : Object.keys(this.gridData[0]).filter(function(x) { return x !== rowIdId })
      },
      orderedHeaders() {
          const headers = this.rawHeaders, ordering = this.columnOrder
          return !ordering.length ? headers : ordering
      },
      filteredHeaders() {
          const hiddenCols = this.hiddenColumns
          return this.orderedHeaders.filter(function(x) { return !hiddenCols.includes(x) })
      },
      headers() {
          const alias = this.aliases
          return this.filteredHeaders.map(function(x) {
              return alias[x] || x
          })
      },
      rawRows() {
          // every row needs id
          if (!this.gridData?.length)
            return []
          
          this.generateRowId(this.gridData[0])
          
          let rowId = -1
          const rowIdId = this.rowId
          
          function addId(row) {
              row[rowIdId] = ++rowId
              return row
          }

          return this.gridData.map(addId)
      },
      filteredRows() {
          const phrases = this.searchPhrases
          let rows = this.rawRows

          console.log('phrases', phrases)

          return rows.filter(function (a) {
              for (let key of Object.keys(phrases)) {
                  let value = phrases[key]
                  if (value && value.length) {
                      if (!a[key].toLowerCase().includes(value))
                        return false
                  }
              }
              return true
          })
      },
      orderedRows() {
          const sortProp = this.sortProperty, sortDir = this.sortDirection
          let rows = JSON.parse(JSON.stringify(this.filteredRows))

          console.log('sort', sortProp)

          return rows.sort(function(a, b) {
              const valueA = a[sortProp], valueB = b[sortProp]
              return (valueA < valueB ? -1 : valueA > valueB ? 1 : 0) * sortDir
          })
          
      },
      sortIndicators() {
          const sortProp = this.sortProperty,
            sortDir = this.sortDirection
          return this.headers.map(function(header) {
              return {
                  isSorting: header === sortProp,
                  direction: sortDir
              }
          })
      }
  },
  watch: {
      stickyHeader(newValue) {
          if (newValue)
            this.$nextTick(this.resized)
      }
  },
  methods: {
      resized() {
          if (this.stickyHeader) {
              for (let index = 0; index < this.filteredHeaders.length; index++) {
                  const headerElement = this.$refs['header-' + index]
                  const refElement = this.$refs['data-' + index]
                  if (headerElement && refElement)
                    headerElement.style.width = refElement.offsetWidth.toString() + 'px'
              }
          }
      },
      rowSelected(row) {
          this.$emit('selected', row)
      },
      generateRowId(object) {
          let i = 1, id = '_id'
          while (object[id]) {
              id = '_'.repeat(++i) + 'id'
          }
          this.rowId = id
      },
      orderBy(header) {
          const index = this.headers.indexOf(header)
          const realHeader = this.filteredHeaders[index]
          if (this.sortProperty === realHeader) {
              this.sortDirection *= -1
          }
          else {
              this.sortProperty = realHeader
              this.sortDirection = 1
          }
      },
      startSearch(header) {
          this.searchInput = this.searchPhrases[header]
          this.searchField = header
          const self = this
          this.$nextTick(function() {
              self.$refs.searchInput.focus()
              self.$refs.searchInput.select()
          })
      },
      stopSearch() {
          if (this.searchField) {
              if (this.searchInput && this.searchInput.length) {
                let phrase = {}
                phrase[this.searchField] = this.searchInput.toLowerCase()
                this.searchPhrases = Object.assign({}, this.searchPhrases, phrase)
              }
              else {
                  let phrase = this.searchPhrases
                  delete phrase[this.searchField]
                  this.searchPhrases = phrase
              }
          }
          
          this.searchInput = ''
          this.searchField = null
      },
      toggleHiddenColumn(header) {
          if (this.hiddenColumns.includes(header)) {
              this.hiddenColumns.splice(this.hiddenColumns.indexOf(header), 1)
          } else {
              this.hiddenColumns.push(header)
          }

          this.saveSettings()
      },
      setAlias(header, value) {
          if (value.length) {
              let alias = {}
              alias[header] = value
              this.aliases = Object.assign({}, this.aliases, alias)
          }
          else {
              let alias = this.aliases
              delete alias[header]
              this.aliases = alias
          }

          this.saveSettings()
      },
      moveHeader(header, index, newIndex) {
          console.log(header)
          this.columnOrder.splice(newIndex, 0, this.columnOrder.splice(index, 1)[0]);
      },
      loadSettings() {
          const obj = JSON.parse(localStorage.getItem('settings_' + this._uid) || '{"hiddenColumns":[],"aliases":{},"columnOrder":[]}')
          this.hiddenColumns = obj.hiddenColumns || []
          this.aliases = obj.aliases || {}
          this.columnOrder = obj.columnOrder || []
          this.stickyHeader = obj.stickyHeader || false
      },
      saveSettings() {
          const obj = {
              hiddenColumns: this.hiddenColumns,
              aliases: this.aliases,
              columnOrder: this.columnOrder,
              stickyHeader: this.stickyHeader
          }
          localStorage.setItem('settings_' + this._uid, JSON.stringify(obj))
      },
      toggleOptions() {
          if (!this.columnOrder.length)
            this.columnOrder = this.rawHeaders
          this.showOptions = !this.showOptions
      }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
.data-grid-container {
    height: 100%;
}
.data-grid-header {
    text-align: left;
}
.data-grid-options-toggle {
    background-color: slategray;
    color: white;
    cursor: pointer;
    outline: none;
}
.data-grid-options {
    position: absolute;
    right: 0;
    max-height: 75%;
    z-index: 9999;
    background-color: slategray;
    border: 1px ridge white;
    text-align: left;
    padding: 1rem;
    font-family: consolas;
    font-size: smaller;
    overflow: auto;
}
.data-grid-options div {
    padding: 0.5rem;
    border-top: 1px solid white;
}
.data-grid-options input {
    border-radius: 5px;
    padding: 3px;
    outline: none;
}
.data-grid-options input[type=checkbox] {
    cursor: pointer;
}
.data-grid-options button {
    border: none;
    outline: none;
    margin-right: 3px;
    background-color: lightsalmon;
    cursor: pointer;
}
.data-grid-table-container {
    height: 90%;
    overflow: auto;
}
.data-grid {
    text-align: left;
    display: table;
    width: 100%;
}
.data-grid-column-header {
    display: table-row;
    height: 3rem;
    overflow: hidden;
}
.data-grid-column-header > div {
    position: relative;
    display: table-cell;
    font-weight: bold;
    background-color: slategray;
    white-space: nowrap;
    line-height: 3;
    border-right: 2px ridge dimgray;
    box-sizing: border-box;
}
.data-grid-column-header > div:last-child {
    border-right: none;
}
.data-grid-row {
    display: table-row;
}
.data-grid-row:nth-child(even) {
    background-color: lightblue;
}
.data-grid-row > div {
    display: table-cell;
    padding: 3px;
}
.data-grid-footer {
    padding: 0 15px;
}
span {
    padding: 5px;
}
.header-text {
    margin-left: 1rem;
}
.text-input:hover {
    cursor: pointer;
    text-decoration: underline;
}
.sort-indicator {
    position: absolute;
    left: 3px;
    padding: 0;
    color: white;
    cursor: pointer;
}
.sort-indicator-none::before {
    content: '<>';
    display: block;
}
.sort-indicator-up::before {
    content: '^';
    display: block;
}
.sort-indicator-down::before {
    content: '^';
    display: block;
    transform: rotate(180deg);
}
.drag-resize {
    float: right;
    width: 3px;
    height: 3rem;
    cursor: ew-resize;
}
.drag-resize:hover {
    filter: brightness(115%);
}
h1 {
    margin: 3px;
    font-size: small;
}
.bold {
    font-weight: bold;
}
.left {
    float: left;
}
.right {
    float: right;
}
.sticky .data-grid-column-header {
    position: fixed;
}
.sticky .margin-top {
    height: 3rem;
}
.clearfix::after {
  content: "";
  clear: both;
  display: table;
}
</style>
