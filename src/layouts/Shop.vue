<template>
  <q-layout view="lHh Lpr lFf">
    <q-header reveal elevated>
      <q-toolbar>
        <q-btn round flat disable @click="goback"/>
        <q-toolbar-title class="row justify-center">
          <span>LITE<b>X</b> Store</span>
        </q-toolbar-title>
        <MenuBtn></MenuBtn>
      </q-toolbar>
      <!-- <OrderStatusBar :status="current.status" pay="pay()" cancel="cancelOrder()" refresh="" /> -->
      <q-tabs>
        <q-route-tab v-for="(category, index) in categorys" :key="index" exact
           :name="getRouter(category.categoryId)" :to="getRouter(category.categoryId)" :label="category.categoryDes" />
      </q-tabs>
    </q-header>
    <q-footer>
      <q-toolbar class="bg-secondary text-white row">
        <q-toolbar-title class="col-6">
          <small> 金额：</small>
          <small :class="[loading ? 'text-grey' : 'text-amber']"> {{ loading ? '加载中..' :  price}} </small>
        </q-toolbar-title>
        <q-separator dark vertical inset />

        <q-btn-dropdown class="col" flat :label="tokens[selected].symbol || '选择币种'">
          <q-list separator>
            <!-- :clickable="token.status === 1" -->
            <q-item class="q-pa-none" v-for="(token, index) in tokens" clickable v-close-popup
              :key="index"  :active="index === selected"
              @click="selectToken(index)">
              <token-item :key="index" :token = "token" @deposit="deposit(token)" @withdraw="withdraw(token)"/>
            </q-item>
          </q-list>
        </q-btn-dropdown>
        <q-separator dark vertical inset />
        <q-btn flat class="col-3 q-pa-sm" label="支付" color="white" @click="placeOrder()" />
      </q-toolbar>
    </q-footer>
    <q-page-container>
      <router-view />
    </q-page-container>

    <confirm-pay-model/>
    <d-remind-model/>
    <pre-dposit-model/>
    <dposit-model/>
    <withdraw-model/>
    <w-remind-model/>
    <deposit-token-model/>
    <order-details-model/>

  </q-layout>
</template>

<script>
import { mapState } from 'vuex'
// import OrderStatusBar from '../components/OrderStatusBar.vue'
import { TokenItem } from '../components/item'
import { ConfirmPayModel, DRemindModel, PreDpositModel, DpositModel, WithdrawModel, WRemindModel, DepositTokenModel, OrderDetailsModel } from '../components/modal'
import MenuBtn from '../components/menu/MenuBtn'
import { getAccount, getNetwork, getRouter, isCurrentUser, getShowToken, toDecimal, mathCeil } from '../utils/helper'
import { Preferences, PrefKeys } from '../utils/preferences'
import Api from '../constants/interface'

// import VConsole from 'vconsole'
// // eslint-disable-next-line no-new
// new VConsole()

// OrderStatusBar
export default {
  name: 'MyLayout',
  components: {
    MenuBtn, TokenItem, ConfirmPayModel, DRemindModel, PreDpositModel, DpositModel, WithdrawModel, WRemindModel, DepositTokenModel, OrderDetailsModel
  },
  data () {
    return {}
  },
  computed: {
    ...mapState('config', {
      duration: 'duration',
      tokens: 'tokens',
      selected: 'selected',
      categorys: 'categorys',
      isInitL2: 'isInitL2',
      account: 'account'
    }),
    ...mapState('sku', {
      info: 'info',
      selectGoods: 'selectGoods'
    }),
    ...mapState('pn', {
      loading: 'loading',
      price: 'price'
    }),
    ...mapState('order', {
      current: 'current'
    }),
    ...mapState('channel', {
      channel: 'channel'
    }),
    phone: function () {
      return this.info.phone
    }
  },
  watch: {
    $route: {
      deep: true,
      handler: function (newVal, oldVal) {
        const { path } = newVal
        switch (path) {
          case '/shop/phone':
            break
          case '/shop/gas':
          case '/shop/vip':
            this.$router.go(-1)
            this.$q.notify({ message: '即将上线, 敬请期待...', position: 'top', color: 'positive', timeout: this.duration })
            break

          default:
            break
        }
      }
    },
    isInitL2: function (newValue, oldValue) {
      if (!this.isInitL2) return
      this.$store.dispatch('config/getOnchainBalance')
      this.$store.dispatch('config/getBalance')
      this.$store.dispatch('config/getChannelInfo')
    },
    channel: function (newValue, oldValue) {
      const { isUpdate = false } = this.channel
      if (isUpdate) {
        this.$store.dispatch('config/getChannelInfo')
      } else {
        this.$store.commit('config/syncChannelStatus', { channel: this.channel })
      }
    }
  },
  methods: {
    getRouter,
    getAccount,
    getNetwork,
    isCurrentUser,
    getShowToken,
    toDecimal,
    mathCeil,
    goback: function () {
      this.$router.go(-1)
    },
    selectToken: function (index) {
      this.$store.commit('config/updateSelected', { index })
      if (!this.isInitL2) return
      this.$store.dispatch('pn/updatePrice')
    },
    placeOrder: function () {
      this.$store.dispatch('order/placeOrder', { phone: this.phone })
    },
    deposit: function (token) {
      console.log('=============【deposit】=======================')
      const { address, symbol } = token
      this.$store.dispatch('channel/preDeposit', { address, symbol })
    },
    withdraw: function (token) {
      console.log('=============【withdraw】=======================')
      const { address, symbol } = token
      this.$store.dispatch('channel/preWithdraw', { address, symbol })
    }
  },
  created: function () {
    this.$store.dispatch('config/getConfigs')

    window.addEventListener('load', async () => {
      console.log('=============load=======================')
      const account = await this.getAccount()
      this.$store.commit('config/update', { account: account.toLowerCase() })

      this.$socket && this.$socket.emit(Api.SOCKET_CONNECT, JSON.stringify({ address: account }))
      Preferences.setItem(PrefKeys.USER_ACCOUNT, account.toLowerCase())
      this.$store.dispatch('config/register')
      this.$store.dispatch('config/initLayer2')

      window.ethereum.on('accountsChanged', (accounts) => {
        console.log('=============【切换 账号】=======================')
        window.location.reload(true)
      })
      window.ethereum.on('networkChanged', function (netId) {
        console.log('=============【切换 netId】=======================')
      })
    })
  },
  sockets: {
    connect: function () {
      this.account && this.$socket.emit(Api.SOCKET_CONNECT, JSON.stringify({ address: this.account }))
    },
    privateMsg: function (res) {
      this.$store.commit('order/depositRes', res)
    }
  },
  mounted: async function () {
    this.$layer2.on('TokenApproval', (err, res) => {
      console.log('===========TokenApproval=========================')
      console.log('TokenApproval from L2', err, res)
      console.log('===========TokenApproval=========================')
      // amount: BigNumber {_hex: "0x02ba7def3000", _ethersType: "BigNumber"}
      // token: "0x3052c3104c32e666666fbef3a5ead4603747ea83"
      // txhash: "0x2c9e8309fdb0a6f20a740ce738b325fa9cebc8da6116626ce289bb4fcf27d8c2"
      // user: "0xb5538753F2641A83409D2786790b42aC857C5340"
      const { user, token } = res
      let { amount } = res
      amount = amount.toString()
      if (!this.isCurrentUser(user)) return
      this.$store.dispatch('config/getBalance')

      const cToken = this.getShowToken(token, this.tokens)
      const { symbol, decimal, float } = cToken
      let value = this.toDecimal({ amount, decimal })
      value = this.mathCeil({ decimal: value, float })
      const message = '成功授权' + ' ' + value + ' ' + symbol
      this.$q.notify({ message, position: 'top', color: 'positive', timeout: this.duration })
      this.$store.dispatch('channel/confirmDeposit', { amount, address: token })

      // this.$store.dispatch('config/getChannelInfo')
    })
    this.$layer2.on('Deposit', (err, res) => {
      console.log('===========Deposit=========================')
      console.log('Deposit from L2', err, res)
      console.log('===========Deposit=========================')
      // amount: '3000000000000'
      // channelID: '0xb06dff751b64958cccbefbfe3b8055b19c865c9e7e478b29d7c81570d4db649e'
      // token: '0x3052c3104c32e666666fBEf3A5EAd4603747eA83'
      // totalDeposit: '3000000000000'
      // user: '0xb5538753F2641A83409D2786790b42aC857C5340'
      const { user, amount, token } = res
      if (!this.isCurrentUser(user)) return
      this.$store.dispatch('config/getOnchainBalance')
      this.$store.dispatch('config/getBalance')

      const cToken = this.getShowToken(token, this.tokens)
      const { symbol, decimal, float } = cToken
      let value = this.toDecimal({ amount, decimal })
      value = this.mathCeil({ decimal: value, float })
      const message = '成功充值' + ' ' + value + ' ' + symbol
      this.$q.notify({ message, position: 'top', color: 'positive', timeout: this.duration })

      this.$store.dispatch('config/getChannelInfo')
    })
    this.$layer2.on('Withdraw', (err, res) => {
      console.log('===========Withdraw=========================')
      console.log('Withdraw from L2', err, res)
      console.log('============Withdraw========================')
      // amount: '30000000000000'
      // balance: '0'
      // token: '0x3052c3104c32e666666fBEf3A5EAd4603747eA83'
      // totalWithdraw: ''
      // txhash: '0xc7a7ab6913676db5b2dfc6ec4989c2cace63c0bf3ad6f9a37def85c589f39c41'
      // user: '0xb5538753F2641A83409D2786790b42aC857C5340'
      const { user, amount, token } = res
      if (!this.isCurrentUser(user)) return
      this.$store.dispatch('config/getOnchainBalance')
      this.$store.dispatch('config/getBalance')

      const cToken = this.getShowToken(token, this.tokens)
      const { symbol, decimal, float } = cToken
      let value = this.toDecimal({ amount, decimal })
      value = this.mathCeil({ decimal: value, float })
      const message = '成功提现' + ' ' + value + ' ' + symbol
      this.$q.notify({ message, position: 'top', color: 'positive', timeout: this.duration })

      this.$store.dispatch('config/getChannelInfo')
    })
    this.$layer2.on('WithdrawUnlocked', (err, res) => {
      console.log('=============WithdrawUnlocked=======================')
      console.log('WithdrawUnlocked from L2', err, res)
      console.log('=============WithdrawUnlocked=======================')
      // amount: '3000000000000'
      // token: '0x3052c3104c32e666666fBEf3A5EAd4603747eA83'
      // user: '0xb5538753F2641A83409D2786790b42aC857C5340'
      const { user, amount, token } = res
      if (!this.isCurrentUser(user)) return
      const cToken = this.getShowToken(token, this.tokens)
      const { symbol, decimal, float } = cToken
      let value = this.toDecimal({ amount, decimal })
      value = this.mathCeil({ decimal: value, float })
      const message = '提现' + ' ' + value + ' ' + symbol + '请求已取消'
      this.$q.notify({ message, position: 'top', color: 'positive', timeout: this.duration })

      this.$store.dispatch('config/getChannelInfo')
    })
    this.$layer2.on('Transfer', (err, res) => {
      console.log('===========Transfer=========================')
      console.log('Transfer from L2', err, res)
      console.log('===========Transfer=========================')
      this.$store.dispatch('config/getBalance')
      this.$store.commit('order/updateShowOrderDModel', { open: true })
      // TODO 更新通道余额
      // TODO 更新钱包余额
      // additionalHash: '0xf569983d1df7db3e9b14014373e7db838cc7c564cae929d9e0afd8b3f7da9ea1'
      // amount: '57900111191373532'
      // channelID: '0x1cd3a84f17cd16723d7de8baa7295fe4fe02bdc3c6143c12f2bf055ed71b2a8e'
      // nonce: '1'
      // token: '0x0000000000000000000000000000000000000000'
      // user: '0xb5538753f2641a83409d2786790b42ac857c5340'
      // userSignature: '0x58f687a2211b4bef1a30921d9f69eb2aac695773f8dd1c392422b880fb2cb46a1b3134a723a4ec5ce1755f2ae410d9748fce1580df55565aad20b1add45b978e1b'
      // userTransferAmount: '57900111191373532'
    })
  }
}
</script>

<style>
</style>

  // test1: function (data) {
  //   console.log('==============test1======================')
  //   console.log(data)
  //   console.log('==============test1======================')
  // },
  // test2: function (data) {
  //   // orderId ==> order details
  //   // type => msg =>(账户 100￥话费 已到账)
  //   console.log('==============test2======================')
  //   console.log(data)
  //   console.log('==============test2======================')
  // }
