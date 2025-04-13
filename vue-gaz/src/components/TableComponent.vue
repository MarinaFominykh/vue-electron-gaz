<template lang="">
  <div class="table-wrapper">
    <table class="table">
      <thead>
        <tr>
          <th class="th" v-for="header in headers" :key="header">{{ header }}</th>
        </tr>
      </thead>
      <tbody>

        <tr
          class="tr__danger"
          v-for="(row, index) in tableData"
          :key="index"
          :style="{ backgroundColor: getCellColor(row.id) }"
        >
          <td class="td" v-for="(value, key) in row" :key="key">
            {{ value }}
          </td>
        </tr>
      </tbody>
    </table>
  </div>
</template>

<script>
export default {
  props: {
    headers: {
      type: Array,
      required: true,
    },
    tableData: {
      type: Array,
      required: true,
    },
    matchObjectIds: {
      type: Array,
      required: true,
    },
  },
  data() {
    return {
      colorMap: {},
    }
  },
  methods: {
    getRandomColor() {
      let letters = '0123456789ABCDEF'
      let color = '#'
      for (let i = 0; i < 6; i++) {
        color += letters[Math.floor(Math.random() * 16)]
      }
      return color
    },
    getCellColor(key) {

      for (const pair of this.matchObjectIds) {
        if (pair.includes(key)) {

          if (!this.colorMap[pair]) {
            this.colorMap[pair] = this.getRandomColor()
          }
          return this.colorMap[pair]
        }
      }
      return 'transparent'
    },
  },
}
</script>

<style scoped>
.table-wrapper {
  width: 100%;
  max-height: 80vh;
  overflow: auto;
  margin: 0 auto;
}

.table {
  border-collapse: collapse;
  width: 100%;
  margin-top: 20px;
}

.th,
.td {
  border: 1px solid #ddd;
  padding: 8px;
  text-align: left;
}

.th {
  background-color: #f2f2f2;
}

.tr__danger {}
</style>
