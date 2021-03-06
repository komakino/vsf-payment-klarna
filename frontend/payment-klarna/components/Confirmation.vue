<template>
  <div class="confirmation">
    <div id="klarna-render-confirmation" />
    <div v-if="confirmation.loading">
      <loading-spinner />
    </div>
    <div v-if="confirmation.snippet" v-html="confirmation.snippet" /> <!-- eslint-disable-line vue/no-v-html -->
  </div>
</template>

<script>
import { mapGetters } from 'vuex'
import qs from 'qs'
import { isServer } from '@vue-storefront/core/helpers'
import LoadingSpinner from './LoadingSpinner.vue'

export default {
  name: 'KlarnaConfirmation',
  computed: {
    ...mapGetters({
      confirmation: 'kco/confirmation',
      storageTarget: 'kco/storageTarget'
    })
  },
  components: {
    LoadingSpinner
  },
  async mounted () {
    if (!isServer) {
      const queryString = this.$route.fullPath.replace(this.$route.path, '')
      let {sid} = qs.parse(queryString, { ignoreQueryPrefix: true })
      const storageTarget = this.storageTarget.replace('/id', '/confirmation/' + sid)
      if (!sid) {
        return
      }
      this.$bus.$emit('kco-order-confirmation', { orderId: sid })
      const result = await this.$store.dispatch('kco/confirmation', { sid })
      this.$bus.$emit('checkout-do-placeOrder', result)
      this.$store.dispatch('cart/clear')
      if (result.merchant_data) {
        if (!localStorage.getItem(storageTarget)) {
          this.$bus.$emit('kco-merchant-data', {
            merchantData: JSON.parse(result.merchant_data),
            result
          })
          localStorage.setItem(storageTarget, 'sent')
        }
        this.$store.dispatch('kco/resetMerchantData')
      }
      const checkboxes = result.merchant_requested && result.merchant_requested.additional_checkboxes
      if (checkboxes) {
        const newsletter = checkboxes.find(({id}) => id === 'newsletter_opt_in')
        if (newsletter && newsletter.checked) {
          this.$bus.$emit('newsletter-signup', {
            email: result.billing_address.email
          })
        }
      }
      const { default: postscribe } = await import('postscribe')
      postscribe('#klarna-render-confirmation', this.confirmation.snippet)
    }
  }
}
</script>

<style lang="scss" scoped>
  .confirmation {
    min-height: 580px;
  }
</style>
