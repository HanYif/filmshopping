<template lang='jade'>
  .SelectSeat
    el-row
      el-col.tips(v-bind:span='6')
        .tip.m-5
          img(src="../assets/seat-lock.png")
          span 已选锁定坑
        .tip.m-5
          img(src="../assets/seat-blank.png")
          span 可选空白坑
        .tip.m-5
          img(src="../assets/seat-yue.png")
          span 可选约影坑
        .tip.m-5
          img(src="../assets/seat-choose.png")
          span 选中这坑啦
      el-col.trapezoid(v-bind:span='12')
        div
      el-col.apply-btn-container(v-bind:span='6')
        el-button.apply-btn.font-10.bold(@click='Y' v-show='!yueinfo.phone') 点我申请约影
        el-button.apply-btn.font-10.bold(@click='Y' v-show='yueinfo.phone') 点我取消约影
    .seats.bg-white.p-30(v-if='tickets')
      .left
        .img(v-for='item in tickets.filter(item => { return item.posY <= 4 })' v-bind:key='item' @click='clickTicket(item)' v-bind:class="item._status !== 1 ? 'pointer' : '' ")
          img(src="../assets/seat-lock.png" v-if='item._status === 1')
          img(src="../assets/seat-blank.png" v-if='item._status === 0')
          img(src="../assets/seat-yue.png" v-if='item._status === 2')
          img(src="../assets/seat-choose.png" v-if='item._status === 3')
      .middle
        .img(v-for='item in tickets.filter(item => { return item.posY > 4 && item.posY <= 15})' v-bind:key='item' @click='clickTicket(item)' v-bind:class="item._status !== 1 ? 'pointer' : '' ")
          img(src="../assets/seat-lock.png" v-if='item._status === 1')
          img(src="../assets/seat-blank.png" v-if='item._status === 0')
          img(src="../assets/seat-yue.png" v-if='item._status === 2')
          img(src="../assets/seat-choose.png" v-if='item._status === 3')
      .right
        .img(v-for='item in tickets.filter(item => { return item.posY > 15 })' v-bind:key='item' @click='clickTicket(item)' v-bind:class="item._status !== 1 ? 'pointer' : '' ")
          img(src="../assets/seat-lock.png" v-if='item._status === 1')
          img(src="../assets/seat-blank.png" v-if='item._status === 0')
          img(src="../assets/seat-yue.png" v-if='item._status === 2')
          img(src="../assets/seat-choose.png" v-if='item._status === 3')

    p.leave-msg(v-bind:class="l_msg ? 'show' : 'hide' ") {{'这是约影者的留言：' + l_msg}}
    el-button.buy-btn.font-12.bold(@click='clkBuy') BUY

</template>

<script>
  export default {
    name: 'SelectSeat',
    created () {
      this.fetchData()
    },
    data () {
      return {
        myTheme: 'light',
        scheduleId: null,
        l_msg: null,
        loading: null,
        tickets: null,
        bindSeatId: null
      }
    },
    methods: {
      fetchData: function () {
        this.bindSeatId = null
        this.$store.commit('applyY', {
          messsage: '',
          phone: ''
        })
        let _loading = this.$loading({fullscreen: true})
        this.loading = true
        this.tickets = null
        this.scheduleId = this.$route.path.replace(/[^\d]/g, '')
        this.$http.get('/api/schedule/' + this.scheduleId).then((resp) => {
          this.$http.get('/api/movie/' + resp.data.movieId).then(mDetail => {
            this.$http.get('/api/cinema/hotcinemas').then(cDetail => {
              console.log('rsp', resp.data)
              console.log('mrsp', mDetail.data)
              console.log('crsp', cDetail.data)
              let cinemaName = ''
              let movieName = mDetail.data.nameCn
              cDetail.data.forEach(c => {
                console.log('cmopare', resp.data.cinemaId, c)
                if (resp.data.cinemaId === c.id) {
                  cinemaName = c.name
                  console.log('cinemaName', c.name)
                }
              })
              console.log('set s info', {
                startTime: resp.data.startTime,
                endTime: resp.data.endTime,
                cinemaName: cinemaName,
                movieName: movieName
              })
              this.$store.commit('setScheduleInfo', {
                startTime: resp.data.startTime,
                endTime: resp.data.endTime,
                cinemaName: cinemaName,
                movieName: movieName
              })
              resp.data.tickets.forEach((item) => {
                item._status = item.status
              })
              this.loading = false
              _loading.close()
              this.tickets = resp.data.tickets
            })
          })
        })
      },
      clickTicket: function (item) {
        // 1:锁定 0:可选 2:可约 3:选中
        if (item._status === 0) {
          if (this.yueinfo.phone) {
            this.autoSelectSeat(item)
          } else {
            this.bindSeatId = [item.id, -1]
          }
          return
        }
        if (item._status === 2) {
          if (this.yueinfo.phone) {
            return
          }
          this.l_msg = item.message
          item._status = 3
          this.bindSeatId = [item.id, -1]
          return
        }
        if (item._status === 3) {
          if (this.yueinfo.phone) {
            this.bindSeatId.forEach((i) => {
              let target = this.tickets.find(item => {
                return i === item.id
              })
              target._status = target.status
            })
          } else {
            item._status = item.status
            this.l_msg = ''
          }
          return
        }
      },
      Y: function () {
        if (this.yueinfo.phone) {
          this.$message('已取消约影')
          this.$store.commit('applyY', {
            messsage: '',
            phone: ''
          })
        } else {
          this.$store.commit('toggleDiglog', 'Y')
        }
        this.bindSeatId = null
      },
      updateBindSeatId: function (newV, oldV) {
        if (oldV && this.tickets) {
          oldV.forEach((id) => {
            if (id !== -1) {
              let target = this.tickets.find(t => {
                return t.id === id
              })
              target._status = target.status
            }
          })
        }
        if (newV && this.tickets) {
          newV.forEach((id) => {
            if (id !== -1) {
              this.tickets.find(t => {
                return t.id === id
              })._status = 3
            }
          })
        }
      },
      autoSelectSeat: function (item, index) {
        var bindItem = null
        if (item.posY === 4 || item.posY === 15 || item.posY === 19) {
          bindItem = this.tickets.find(t => {
            return t.posX === item.posX && t.posY === item.posY - 1
          })
        } else {
          bindItem = this.tickets.find(t => {
            return t.posX === item.posX && t.posY === item.posY + 1
          })
        }
        if (bindItem._status === 1 || bindItem._status === 2) {
          return
        }
        this.bindSeatId = [item.id, bindItem.id]
      },
      clkBuy: function () {
        let seatInfo = this.bindSeatId
        let tickets = []
        if (!seatInfo) {
          this.$alert('请先选择座位！')
          return
        }
        this.bindSeatId.forEach(id => {
          let ticket = this.tickets.find(item => {
            return item.id === id
          })
          if (ticket) {
            tickets.push({
              posX: ticket.posX,
              posY: ticket.posY
            })
          }
        })
        console.log('ticktes', tickets)
        this.$store.commit('setOrder', seatInfo)
        this.$store.commit('setTickets', tickets)
        console.log('s s tickets', this.tickets__)
        this.$store.commit('toggleDiglog', 'Pay')
      },
      rmSeatIfCancelY: function () {
        if (this.yueinfo.phone === '') {
          this.bindSeatId = null
        }
      }
    },
    computed: {
      yueinfo () {
        return this.$store.getters.getYInfo
      },
      updateSeats () {
        return this.$store.getters.getUpdateSeats
      },
      tickets__ () {
        return this.$store.getters.getTickets
      }
    },
    watch: {
      // 如果路由有变化，会再次执行该方法
      '$route': 'fetchData',
      updateSeats: 'fetchData',
      bindSeatId: 'updateBindSeatId'
    }
  }
</script>

<style lang="sass" scoped>
$bg-color: #fba214

.SelectSeat
  min-width: 1000px
  min-height: 500px
  .el-row
    .tips
      .tip
        color: white
        font-weight: bold
        img
          width: 30px
        span
          position: relative
          bottom: 10px
          left: 5px
    .trapezoid
      div
        height: 0;
        border-bottom: 100px solid #232323;
        border-left: 250px solid transparent;
        border-right: 250px solid transparent;
    .apply-btn-container
      position: relative
      top: 20px
      .apply-btn
        background-color: black
        color: $bg-color
        border: 0
  .seats
    display: flex
    justify-content: space-around
    width: 1000px
    margin: 10px auto
    .img
      display: inline-block
      width: 34px
      margin: 2.5px 0px
      img
        width: 100%
    .left
      width: 140px
      height: 100%
    .middle
      width: 400px
      height: 100%
    .right
      width: 140px
      height: 100%

  .buy-btn
    background-color: black
    color: $bg-color
    border: 0
  .leave-msg

.pointer
  cursor: pointer
.show
  visibility: visible
.hide
  visibility: hidden
</style>

