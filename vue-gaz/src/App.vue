<template>
  <main class="app">
    <h1 class="app__title">Поиск совпадений по объектам газификации</h1>
    <div class="app__buttons">
      <el-upload
        class="upload"
        :limit="1"
        :auto-upload="false"
        :on-change="handleChange"
        :on-remove="handleRemove"
      >
        <el-button type="primary">Выберите файл</el-button>
      </el-upload>
      <el-select
        v-model="selectedMatch"
        placeholder="выберите фильтр"
        size="large"
        class="select"
        clearable
        :disabled="!filteredObjects.length"
      >
        <el-option
          v-for="item in options"
          :key="item.value"
          :label="item.label"
          :value="item.value"
        />
      </el-select>
    </div>

    <div v-if="filteredObjects.length">
      <h2 class="app__subtitle">Список объектов газификации</h2>
      <table-component
        :headers="headers"
        :tableData="filteredObjects"
        :matchObjectIds="allObjectsIds"
      />
    </div>
  </main>
</template>

<script>
import * as XLSX from 'xlsx'
import { toRaw } from 'vue'
import { UploadFilled } from '@element-plus/icons-vue'
import { ref } from 'vue'
import TableComponent from './components/TableComponent.vue'

export default {
  components: {
    UploadFilled,
    TableComponent,
  },
  data() {
    return {
      tableData: [],
      headers: [],
      data: [],
      codeMatchIds: [],
      addressMatchIds: [],
      allObjectsIds: [],
      upload: ref(),
      options: [
        { value: 'address', label: 'по адресу' },
        { value: 'code', label: 'по коду' },
        { value: 'all', label: 'все совпадения' },
      ],
      selectedMatch: '',
    }
  },
  methods: {
    handleChange(file) {
      if (!file) return
      const reader = new FileReader()
      reader.onload = (e) => {
        const data = new Uint8Array(e.target.result)
        const workbook = XLSX.read(data, { type: 'array' })

        // Берем первую страницу
        const firstSheetName = workbook.SheetNames[0]
        const worksheet = workbook.Sheets[firstSheetName]

        // Конвертируем в JSON
        const jsonData = XLSX.utils.sheet_to_json(worksheet)

        if (jsonData.length) {
          this.headers = Object.keys(jsonData[0])
          this.tableData = jsonData
          this.searchForMatches()
        }
      }
      reader.readAsArrayBuffer(file.raw)
    },
    handleRemove() {
      this.headers = []
      this.tableData = []
      this.codeMatchIds = []
      this.addressMatchIds = []
      this.allObjectsIds = []
      this.selectedMatch = ''
    },
    searchForMatches() {
      const map = new Map()
      const arrHasNotKeyOrValue = []
      const rawData = toRaw(this.tableData)
      const mapNotValidAddress = new Map()

      rawData.forEach((element) => {
        const location = element.location
        if (!map.has(location)) {
          map.set(location, [])
        }
        map.get(location)?.push({ id: element.id, address: element.address, code: element.code })
      })

      map.forEach((value, key, map) => {
        const mapAddressGroups = new Map()
        const mapCodeGroups = new Map()

        if (value.length > 1) {
          value.forEach((item) => {
            if (!mapCodeGroups.has(item.code)) {
              mapCodeGroups.set(item.code, [])
            }
            mapCodeGroups.get(item.code)?.push(item)
            if (item.address && key) {
              try {
                const entries = String(item.address)
                  .trim()
                  .toLowerCase()
                  .split(/[, ]+/)
                  .map((part) => part.trim())
                if (entries.length === 3) {
                  const [_, street, house] = entries
                  const key = `${street}-${house}`
                  if (!mapAddressGroups.has(key)) {
                    mapAddressGroups.set(key, [])
                  }
                  mapAddressGroups.get(key)?.push(item)
                } else {
                  if (!mapNotValidAddress.has(key)) {
                    mapNotValidAddress.set(key, [])
                  }
                  mapNotValidAddress.get(key)?.push(item)
                }
              } catch (err) {
                console.log('error item', item)
              }
            } else {
              arrHasNotKeyOrValue.push(item.id)
            }
          })
        }

        const addressGroupsArray = Array.from(mapAddressGroups.values()).filter(
          (ids) => ids.length > 1,
        )

        if (addressGroupsArray.length > 0) {
          //this.matchesValidAddress.push({ location: key, intersections: addressGroupsArray })
          this.addressMatchIds.push(...addressGroupsArray.map((item) => item.map((i) => i.id)))
          this.allObjectsIds.push(...addressGroupsArray.map((item) => item.map((i) => i.id)))
        }
        const codeGroupsArray = Array.from(mapCodeGroups.values()).filter((item) => item.length > 1)
        if (codeGroupsArray.length > 0) {
          // this.matchesCode.push({ location: key, intersections: codeGroupsArray })
          this.codeMatchIds.push(...addressGroupsArray.map((item) => item.map((i) => i.id)))
          this.allObjectsIds.push(...addressGroupsArray.map((item) => item.map((i) => i.id)))
        }
      })
      this.matchesAllAddress = this.findSimilarAddresses(map)
      this.allObjectsIds.push(
        ...this.matchesAllAddress.map((item) => item.intersections.map((item) => item.id)),
      )
    },

    findSimilarAddresses(locationMap) {
      const similarPairs = []

      for (const [locationName, addresses] of locationMap.entries()) {
        for (let i = 0; i < addresses.length; i++) {
          for (let j = i + 1; j < addresses.length; j++) {
            const address1 = String(addresses[i].address)
            const address2 = String(addresses[j].address)
            const similarity = address1.includes(address2)

            if (similarity) {
              similarPairs.push({
                location: locationName,
                intersections: [
                  { id: addresses[i].id, address: address1 },
                  { id: addresses[j].id, address: address2 },
                ],
                // address1: { id: addresses[i].id, address: address1 },
                // address2: { id: addresses[j].id, address: address2 },
                // similarity: similarity,
              })
            }
          }
        }
      }

      return similarPairs
    },
  },
  computed: {
    filteredObjects() {
      let uniqueAllObjectsIds = toRaw(this.allObjectsIds)
        .filter(
          (item, index) =>
            index ===
            this.allObjectsIds.findIndex((other) => JSON.stringify(other) === JSON.stringify(item)),
        )
        .flat(1)
      switch (this.selectedMatch) {
        case 'address':
          return this.tableData.filter((item) => this.addressMatchIds.flat(1).includes(item.id))
        case 'code':
          return this.tableData.filter((item) => this.codeMatchIds.flat(1).includes(item.id))
        case 'all':
          return this.tableData.filter((item) => uniqueAllObjectsIds.includes(item.id))
        default:
          return this.tableData
      }
    },
  },
}
</script>

<style>
body {
  margin: 0;
  background-color: #f3f3f2;
}

.app {
  max-width: 1280px;
  margin: 0 auto;
  padding: 20px;
  font-family: 'Inter', sans-serif;
  font-optical-sizing: auto;
  font-weight: 400;
  font-style: normal;
  font-size: 16px;
}
.app__title {
  text-align: center;
  font-size: 24px;
}
.app__subtitle {
  text-align: center;
  font-size: 18px;
}
.app__buttons {
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.upload {
  width: 350px;
}
.select {
  width: 240px;
  font-family: 'Inter', sans-serif;
  font-optical-sizing: auto;
  font-weight: 400;
  font-style: normal;
}
</style>
