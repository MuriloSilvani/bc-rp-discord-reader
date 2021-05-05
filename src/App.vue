<template>
  <div id="app">

    <label>Dia</label>
    <input v-model="date" type="date">
    <label>De</label>
    <input v-model="minDate" type="datetime-local">
    <label>Até</label>
    <input v-model="maxDate" type="datetime-local">
    <button @click="loadLogs()">Filtrar</button>

    <h4> TODOS OS USUÁRIOS </h4>
    <table>
      <thead>
        <tr>
          <th>Nome</th>
        </tr>
      </thead>
      <tbody>
        <tr
          v-for="(user, index) in users"
          :key="index"
          @click="current = user"
        >
          <td>{{ user.name }}</td>
        </tr>
      </tbody>
    </table>

    <div v-if="current">
      <h4> LOGS DO JOGADOR {{ current.name }} </h4>

      <table>
        <thead>
          <tr>
            <th>Item</th>
            <th>Saldo</th>
          </tr>
        </thead>
        <tbody>
          <tr
            v-for="(det, index) in detailed"
            :key="index"
          >
            <td>{{ det.item }}</td>
            <td>{{ det.value }}</td>
          </tr>
        </tbody>
      </table>

      <table>
        <thead>
          <tr>
            <th>Ação</th>
            <th>Item</th>
            <th>Valor</th>
            <th>Data</th>
          </tr>
        </thead>
        <tbody>
          <tr
            v-for="log in current.data"
            :key="log.id"
          >
            <td>{{ log.action }}</td>
            <td>{{ log.item }}</td>
            <td>{{ log.value }}</td>
            <td>{{ log.date }}</td>
          </tr>
        </tbody>
      </table>
    </div>

    <h4> TODOS OS LOGS </h4>
    <table>
      <thead>
        <tr>
          <th>Nome</th>
          <th>Ação</th>
          <th>Item</th>
          <th>Valor</th>
          <th>Data</th>
        </tr>
      </thead>
      <tbody>
        <tr
          v-for="log in logs"
          :key="log.id"
        >
          <td>{{ log.user }}</td>
          <td>{{ log.action }}</td>
          <td>{{ log.item }}</td>
          <td>{{ log.value }}</td>
          <td>{{ log.date }}</td>
        </tr>
      </tbody>
    </table>
  </div>
</template>

<script>
import axios from 'axios'
import { DateTime } from 'luxon'

export default {
  name: 'App',
  data () {
    return {
      logs: [],
      current: null,
      date: null,
      minDate: null,
      maxDate: null
    }
  },
  computed: {
    users () {
      const users = {}

      this.logs.forEach(o => {
        users[o.user] = {
          user: o.user
        }
      })

      const result = Object.keys(users).map(o => {
        return {
          name: users[o].user
        }
      }).map(o => {
        return {
          ...o,
          data: this.logs.filter(p => p.user === o.name)
        }
      })

      return result
    },
    detailed () {
      if (!this.current) { return [] }
      const data = this.current.data

      const fields = {}

      data.forEach(o => {
        if (fields[o.item]) {
          if (o.action.includes('RETIROU')) {
            fields[o.item] -= Number(o.value)
          } else {
            fields[o.item] += Number(o.value)
          }
        } else {
          fields[o.item] = Number(o.value)
        }
      })

      return Object.keys(fields).map(o => {
        return {
          item: o,
          value: fields[o]
        }
      })
    }
  },
  watch: {
    date (e) {
      if (e) {
        const start = DateTime.fromISO(e).startOf('day')
        const ent = DateTime.fromISO(e).endOf('day')
        this.minDate = `${start.toFormat('yyyy-MM-dd')}T${start.toFormat('HH:mm:ss')}`
        this.maxDate = `${ent.toFormat('yyyy-MM-dd')}T${ent.toFormat('HH:mm:ss')}`
      }
    }
  },
  methods: {
    async loadLogs () {
      const logs = []
      const getLogs = async (before = null) => {
        const response = await axios.get(
          `https://discord.com/api/v9/channels/812467120239673374/messages?limit=100${before ? `&before=${before}` : ``}`,
          {
            headers: {
              // authorization: 'MzA1NDE2NTQxMjUwMjU2OTAw.YI17Vg.KDE25p964jfemEjB4fVcDwx4MtI'
            }
          }
        )
        return response.data
      }
      
      let hasMore = true
      const dataLogs = []
      let id = null
      if (this.minDate && this.maxDate && DateTime.fromISO(this.minDate).toMillis() > DateTime.fromISO(this.maxDate).toMillis()) {
        const aux = this.maxDate
        this.maxDate = this.minDate
        this.minDate = aux
      }
      while (hasMore) {
        const data = await getLogs(id)
        if (data.length > 0) {
          const today = DateTime.local()
          await data.forEach(o => {

            const ifMin = (DateTime.fromISO(o.timestamp).toMillis()) > (this.minDate ? DateTime.fromISO(this.minDate).toMillis() : today.startOf('day').toMillis())
            const ifMax = (DateTime.fromISO(o.timestamp).toMillis()) < (this.maxDate ? DateTime.fromISO(this.maxDate).toMillis() : today.endOf('day').toMillis())

            if (
              ifMin && ifMax
            ) {
              dataLogs.push(o)
            }
          })

          const last = dataLogs[dataLogs.length - 1]
          if (last && id !== last.id) {
            id = last.id

            const ifDate = (DateTime.fromISO(last.timestamp).toMillis()) < (this.minDate ? DateTime.fromISO(this.minDate).toMillis() : today.startOf('day').toMillis())

            if (ifDate) {
              hasMore = false
            }
          } else {
            hasMore = false
          }
        } else {
          hasMore = false
        }
      }

      dataLogs.forEach(o => {
        const data = o.embeds[0].description.split('**')
        logs.push({
          id: o.id,
          user: data[1] + ` ${data[2].replace('Item/Armamento:', '').trim()}`,
          action: o.embeds[0].title.replace('**O Jogador ', '').replace(' um item/arma**', ''),
          item: data[3],
          value: Number(data[5].replace('.', '')),
          date: data[9],
        })
      })

      this.logs = logs
    }
  },
  mounted () {
    this.loadLogs()
  }
}
</script>