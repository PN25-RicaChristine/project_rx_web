<template>
  <div class="row">
    <div class="col-md-6 mx-auto">
      <div class="form-group" v-if="errorMessage !== null">
        <label class="text-danger">{{errorMessage}}</label>
      </div>
      <div class="form-group">
        <label>Account Code</label>
        <div class="input-group">
          <input type="text" v-model="code" class="form-control" />
          <button class="btn btn-primary btn-input" @click="retrieveAccount()">
            <i class="fa fa-search text-white"></i>
          </button>
        </div>
      </div>
      <div class="form-group" v-if="account !== null">
        <label><b>Account Information</b></label>
        <div class="form-group">
          <label>
            Username: {{account.username}}
          </label>
        </div>
        <div class="form-group">
          <label>
            Email: {{account.email}}
          </label>
        </div>
      </div>
      <div class="form-group" v-if="account !== null">
        <div class="form-group">
          <label>Transaction type</label>
          <div class="input-group">
            <select class="form-control form-control-custom" v-model="type">
              <option value="AUTOMATIC">AUTOMATIC</option>
              <option value="MANUAL">MANUAL</option>
            </select>
          </div>
        </div>

        <div class="form-group">
          <label>Currency</label>
          <div class="input-group">
            <select class="form-control form-control-custom" v-model="currency">
              <option v-for="(item, index) in countries"  v-bind:value="item.currency" :key="index">{{item.currency}}</option>
            </select>
          </div>
        </div>

        <div class="form-group">
          <label>Amount</label>
          <div class="input-group">
            <input type="number" v-model="amount" class="form-control form-control-custom" />
          </div>
        </div>

        <div class="form-group">
          <label>Description</label>
          <div class="input-group">
            <input type="text" v-model="description" class="form-control form-control-custom" placeholder="Transfer description" />
          </div>
        </div>

        <div class="form-group" v-if="type === 'AUTOMATIC'">
          <label>From Description</label>
          <div class="input-group">
            <input type="text" v-model="fromDescription" class="form-control form-control-custom" placeholder="Account Transfer description(From)" />
          </div>
        </div>

        <div class="form-group" v-if="transferPayloads.length > 0">
          <label>Payload</label>
          <div class="input-group">
            <select class="form-control form-control-custom" v-model="payload">
              <option :value="item.payload_value" v-for="(item, index) in transferPayloads">{{item.payload_value}}</option>
            </select>
          </div>
        </div>

        <div class="form-group">
          <label>Payload Value</label>
          <div class="input-group">
            <input type="text" v-model="payloadValue" class="form-control form-control-custom" placeholder="Transfer payload value" />
          </div>
        </div>

        <div class="form-group">
          <button class="btn btn-primary pull-right" @click="validation()">Continue</button>
          <!-- <button class="btn btn-primary pull-right" @click="$refs.otpModal.show()">Continue</button> -->
        </div>
       
      </div>
    </div>
    <otp-modal ref="otpModal"></otp-modal>
    <Confirmation
      :title="'Confirmation'"
      :message="'Are you sure you want this transaction as ' + type + '?'"
      ref="confirmation"
      @onConfirm="transfer"
      />
    <IncrementAlert
      :title="alertTitle"
      :message="alertMessage"
      ref="incrementAlert"
      />
  </div>
</template>
<style scoped lang="scss">
@import "~assets/style/colors.scss";

.text-primary{
  color: $primary !important;
}

.input-group{
  margin-bottom: 10px !important;
}

.form-control-custom, .btn-input{
  min-height: 50px !important;
}

.fa{
  padding-right: 0px !important;
}
</style>
<script>
import ROUTER from 'src/router'
import AUTH from 'src/services/auth'
import CONFIG from 'src/config.js'
import COMMON from 'src/common.js'
import COUNTRIES from 'src/countries.js'
import Confirmation from 'src/components/increment/generic/modal/Confirmation.vue'
import IncrementAlert from 'src/components/increment/generic/modal/Alert.vue'
export default {
  mounted(){
    this.retrieveTransferPayloads()
  },
  data(){
    return {
      user: AUTH.user,
      auth: AUTH,
      config: CONFIG,
      code: null,
      amount: 0,
      currency: 'PHP',
      account: null,
      countries: COUNTRIES.list,
      description: null,
      payload: null,
      payloadValue: null,
      type: 'AUTOMATIC',
      fromDescription: null,
      stage: 1,
      opt: null,
      alertMessage: null,
      alertTitle: null,
      errorMessage: null,
      transferPayloads: []
    }
  },
  props: [],
  components: {
    'otp-modal': require('components/increment/generic/otp/OtpInput.vue'),
    Confirmation,
    IncrementAlert
  },
  methods: {
    retrieveTransferPayloads(){
      let parameter = {
        condition: [{
          clause: '=',
          column: 'payload',
          value: 'transfer_payloads'
        }],
        sort: {
          payload_value: 'asc'
        }
      }
      this.APIRequest('payloads/retrieve', parameter).then(response => {
        this.transferPayloads = response.data
      })
    },
    hideModal(){
      this.account = null
      this.code = null
      this.amount = 0
      this.currency = 'PHP'
      this.description = null
      this.payload = null
      this.payloadValue = null
    },
    onContinue(){
      this.$emit('onContinue', {id: this.id})
      this.id = null
      this.hideModal()
    },
    successOTP(otp){
      this.otp = otp
      this.transfer()
    },
    retrieveAccount(){
      if(this.code === null){
        return null
      }
      let parameter = {
        condition: [{
          value: this.code,
          column: 'code',
          clause: '='
        }]
      }
      $('#loading').css({display: 'block'})
      this.APIRequest('accounts/retrieve_accounts', parameter).then(response => {
        $('#loading').css({display: 'none'})
        if(response.data.length > 0){
          this.account = response.data[0]
        }else{
          this.account = null
        }
      })
    },
    showConfirmation(){
      this.$refs.confirmation.show(null)
    },
    validation(){
      this.errorMessage = null
      if(this.amount <= 0){
        this.errorMessage = 'Amount must be greater than 0.'
        return
      }
      if(this.payload === null || this.payloadValue === null || (this.fromDescription === null && this.type === 'AUTOMATIC')){
        this.errorMessage = 'All fields are required.'
        return
      }
      if(this.errorMessage !== null){
        return
      }
      this.showConfirmation()
    },
    transfer(){
      let parameter = {
        from_account_id: this.user.userID,
        from_account_code: this.user.code,
        currency: this.currency,
        account_id: this.account.id,
        account_code: this.account.code,
        payment_payload: this.payload,
        payment_payload_value: this.payloadValue,
        description: this.description,
        from_description: this.fromDescription,
        amount: this.amount,
        otp: this.otp,
        stage: this.stage,
        type: this.type
      }
      $('#loading').css({display: 'block'})
      this.APIRequest('ledger/transfer', parameter).then(response => {
        $('#loading').css({display: 'none'})
        this.stage = response.stage
        if(response.error !== null){
          this.alertTitle = 'Error Message'
          this.alertMessage = response.error
          this.stage = 1
          this.$refs.incrementAlert.show(null)
        }else if(response.stage === 2){
          this.$refs.otpModal.show()
        }else if(response.stage === 3 && response.data === true){
          //
          this.alertTitle = 'Success Message'
          this.alertMessage = 'Your transaction was successfully processed.'
          this.$refs.incrementAlert.show('/admin/transfer')
        }else{
        }
      })
    }
  }
}
</script>
