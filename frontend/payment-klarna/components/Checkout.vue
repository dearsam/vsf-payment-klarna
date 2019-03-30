<template>
  <div class="klarna-checkout" id="klarna-checkout">
    <div id="ref-scripts" ref="scripts" />
    <div id="this-loading" v-if="loading">Loading</div>
    <div id="this-snippet" v-if="snippet" v-html="snippet" />
  </div>
</template>

<script>
import { currentStoreView } from '@vue-storefront/core/lib/multistore'
import config from 'config'
import { mapGetters } from 'vuex'
import {
  post,
  getScriptTagsFromSnippet,
  callApi,
  mapProductToKlarna,
  calculateShippingTaxRate
} from './helpers'

const storageTarget = '@vsf/klarna_order_id'

export default {
  name: 'KlarnaCheckout',
  data () {
    return {
      order: {
        order_lines: [],
        order_amount: 0,
        order_tax_amount: 0
      },
      snippet: null,
      createdOrder: {
        id: ''
      },
      loading: false,
      storeView: currentStoreView()
    }
  },
  async mounted () {
    // The call chain
    this.saveOrderIdToLocalStorage()
    this.order.order_amount = this.grandTotal
    this.order.order_tax_amount = this.taxAmount
    this.addCartItemsToOrder()
    this.configureLocaleAndMerchant()
    await this.upsertOrder()
    this.saveOrderIdToLocalStorage()
  },
  beforeMount () {
    this.$bus.$on('updateKlarnaOrder', this.configureUpdateOrder())
  },
  computed: {
    ...mapGetters({
      cartItems: 'cart/items',
      cartTotals: 'cart/totals',
      shippingInformation: 'cart/shippingInformation'
    }),
    shippingMethodName () {
      return this.cartTotals.find(seg => seg.code === 'shipping').title
    },
    subTotalInclTax () {
      return this.cartTotals.find(seg => seg.code === 'subtotalInclTax').value * 100
    },
    grandTotal () {
      return this.cartTotals.find(seg => seg.code === 'grand_total').value * 100
    },
    taxAmount () {
      return this.cartTotals.find(seg => seg.code === 'tax').value * 100
    }
  },
  methods: {
    addCartItemsToOrder () {
      const orderLines = this.cartItems.map(mapProductToKlarna)
      const shippingTaxRate = calculateShippingTaxRate(this.shippingInformation)
      const unitPrice = this.shippingInformation.platformTotals.base_shipping_incl_tax * 100
      const shippingInformation = {
        type: 'shipping_fee',
        reference: 'SHIPPING REFERENCE HERE',
        name: this.shippingMethodName,
        quantity: 1,
        unit_price: unitPrice,
        tax_rate: shippingTaxRate,
        total_tax_amount: unitPrice * shippingTaxRate,
        total_amount: unitPrice - (unitPrice * shippingTaxRate)
      }
      this.order.order_lines = [...orderLines, shippingInformation]
    },
    configureLocaleAndMerchant () {
      const checkoutOrder = {
        purchase_country: this.storeView.i18n.defaultCountry,
        purchase_currency: this.storeView.i18n.currencyCode,
        locale: this.storeView.i18n.defaultLocale,
        merchant_urls: config.klarna.checkout.merchant
      }
      this.order = { ...this.order, ...checkoutOrder }
    },
    async upsertOrder () {
      let url = config.klarna.endpoint

      const body = {
        order: this.order,
        userAgent: navigator.userAgent
      }
      if (this.getOrderId()) {
        body.orderId = this.getOrderId()
      }
      this.loading = true
      const { result } = await post(url, body)
      const scriptsTags = getScriptTagsFromSnippet(result.snippet)
      this.snippet = result.snippet
      setTimeout(() => {
        Array.from(scriptsTags).forEach(tag => {
          // TODO: Make this work with <script> tag insertion
          (() => {eval(tag.text)}).call(window) // eslint-disable-line
          this.$refs.scripts.appendChild(tag)
        })
        this.createdOrder.id = result.orderId || null
        this.loading = false
      }, 1)
    },
    async configureUpdateOrder () {
      if (!this.createdOrder.id) {
        return
      }
      await this.suspendCheckout()
      this.order.cart.items = {}
      this.addCartItemsToOrder()
      await this.upsertOrder()
      await this.resumeCheckout()
    },
    suspendCheckout () {
      return callApi(api => api.suspend())
    },
    resumeCheckout () {
      return callApi(api => api.resume())
    },
    saveOrderIdToLocalStorage () {
      this.createdOrder.id
        ? localStorage.setItem(storageTarget, this.createdOrder.id)
        : localStorage.removeItem(storageTarget)
    },
    getOrderId () {
      return this.createdOrder.id || localStorage.getItem(storageTarget)
    }
  }
}
</script>