<template>
  <ModalWindow
    :title="$t('SIGN_VERIFY.TITLE_DELEGATE_TRANSACTION')"
    @close="emitCancel"
  >
    <div class="flex flex-col justify-center w-80">
      <InputText
        :is-read-only="true"
        :label="$t('SIGN_VERIFY.ADDRESS')"
        :value="wallet.address"
        class="mt-5"
        name="address"
      />

      <PassphraseInput
        v-if="!wallet.passphrase"
        ref="passphrase"
        v-model="$v.form.passphrase.$model"
        :is-invalid="$v.form.passphrase.$error"
        :address="wallet.address"
        :pub-key-hash="session_network.version"
        class="my-3"
      />

      <InputPassword
        v-else
        ref="password"
        v-model="$v.form.walletPassword.$model"
        :label="$t('TRANSACTION.PASSWORD')"
        :is-required="true"
        class="my-3"
      />

      <InputText
        v-model="$v.form.username.$model"
        :helper-text="error"
        :label="$t('WALLET_DELEGATES.USERNAME')"
        :is-invalid="$v.form.username.$dirty && $v.form.username.$invalid"
        class="mb-5"
        name="username"
      />

      <div :class="{ 'hidden': !showSignedTransaction }">
        <label class="text-theme-page-text-light">
          {{ $t('SIGN_VERIFY.SIGNED_TRANSACTION') }}
          <ButtonClipboard
            :value="signedTransaction"
            class="bg-theme-modal text-theme-page-text-light"
          />
        </label>

        <textarea
          v-model="signedTransaction"
          readonly
          class="bg-theme-modal text-theme-page-text-light"
          rows="8"
          cols="40"
        />
      </div>
    </div>
    <button
      :disabled="$v.form.$invalid"
      class="blue-button mt-5"
      type="button"
      @click="onSignTransaction"
    >
      {{ $t('SIGN_VERIFY.SIGN') }}
    </button>

    <ModalLoader
      :message="$t('ENCRYPTION.DECRYPTING')"
      :visible="showEncryptLoader"
    />
  </ModalWindow>
</template>

<script>
import { InputPassword, InputText } from '@/components/Input'
import { ModalLoader, ModalWindow } from '@/components/Modal'
import { ButtonClipboard } from '@/components/Button'
import { PassphraseInput } from '@/components/Passphrase'
import WalletService from '@/services/wallet'
import Bip38 from '@/services/bip38'
import store from '@/store'

export default {
  name: 'WalletDelegateSignModal',

  components: {
    ButtonClipboard,
    InputPassword,
    InputText,
    ModalLoader,
    ModalWindow,
    PassphraseInput
  },

  props: {
    wallet: {
      type: Object,
      required: true
    }
  },

  data: () => ({
    form: {
      username: '',
      passphrase: '',
      walletPassword: ''
    },
    error: null,
    showEncryptLoader: false,
    signedTransaction: '',
    showSignedTransaction: false
  }),

  computed: {
    messageError () {
      if (this.$v.form.username.$error && this.$v.form.message.minLength) {
        return this.$t('VALIDATION.REQUIRED', [this.$refs['message'].label])
      }
      return null
    }
  },

  methods: {
    async onSignTransaction () {
      if (this.form.walletPassword && this.form.walletPassword.length) {
        this.showEncryptLoader = true

        const dataToDecrypt = {
          bip38key: this.wallet.passphrase,
          password: this.form.walletPassword,
          wif: this.session_network.wif
        }

        const bip38 = new Bip38()
        try {
          const { encodedWif } = await bip38.decrypt(dataToDecrypt)
          this.form.passphrase = null
          this.form.wif = encodedWif
        } catch (_error) {
          this.$error(this.$t('ENCRYPTION.FAILED_DECRYPT'))
        } finally {
          bip38.quit()
        }

        this.showEncryptLoader = false
      }

      this.buildAndSignTransaction()
    },

    async buildAndSignTransaction () {
      try {
        const transactionData = {
          username: this.form.username,
          passphrase: this.form.passphrase,
          wif: this.form.wif,
          networkWif: store.getters['session/network'].wif,
          genesis: true
        }
        const transaction = await this.$client.buildDelegateRegistration(transactionData)
        this.signedTransaction = JSON.stringify(transaction, null, 4)
        this.showSignedTransaction = true
        this.$success(this.$t('SIGN_VERIFY.SUCCESSFULL_SIGN'))
      } catch (error) {
        this.showSignedTransaction = false
        console.log(error)
        this.$error(this.$t('SIGN_VERIFY.FAILED_SIGN'))
      }
    },

    emitCancel () {
      this.$emit('cancel')
    }
  },

  validations: {
    form: {
      username: {
        isValid (value) {
          const validation = WalletService.validateUsername(value)

          if (validation.passes) {
            this.error = null
          } else {
            switch (validation.errors[0].type) {
              case 'empty':
                this.error = this.$t('WALLET_DELEGATES.USERNAME_EMPTY_ERROR')
                break
              case 'maxLength':
                this.error = this.$t('WALLET_DELEGATES.USERNAME_MAX_LENGTH_ERROR')
                break
              default:
                this.error = this.$t('WALLET_DELEGATES.USERNAME_ERROR')
            }
          }

          return validation.passes
        }
      },
      passphrase: {
        isValid (value) {
          if (this.wallet.passphrase) {
            return true
          }

          if (this.$refs.passphrase) {
            return !this.$refs.passphrase.$v.$invalid
          }

          return false
        }
      },
      walletPassword: {
        isValid (value) {
          if (!this.wallet.passphrase) {
            return true
          }

          if (!this.form.walletPassword || !this.form.walletPassword.length) {
            return false
          }

          if (this.$refs.password) {
            return !this.$refs.password.$v.$invalid
          }

          return false
        }
      }
    }
  }
}
</script>
