<template>
  <div>
    <h2>{{ $t('ssh_keys.title') }}</h2>

    <!-- Toggle password vs SSH key authentication -->
    <b-form-group label="{{ $t('ssh_keys.auth_method') }}">
      <b-form-radio-group v-model="authMethod">
        <b-form-radio value="password">{{ $t('ssh_keys.password') }}</b-form-radio>
        <b-form-radio value="keys">{{ $t('ssh_keys.ssh_key') }}</b-form-radio>
      </b-form-radio-group>
      <b-alert variant="warning" v-if="authMethod === 'keys'">
        {{ $t('ssh_keys.warning') }}
      </b-alert>
    </b-form-group>

    <!-- SSH key management section -->
    <div v-if="authMethod === 'keys'">
      <b-button @click="generateKey">{{ $t('ssh_keys.generate') }}</b-button>
      <b-form-file v-model="uploadedKey" :placeholder="$t('ssh_keys.upload_placeholder')"></b-form-file>
      <b-button @click="uploadKey">{{ $t('ssh_keys.upload') }}</b-button>

      <b-table striped hover :items="keys" :fields="fields">
        <template #cell(actions)="data">
          <b-button size="sm" @click="copyKey(data.item.key)">{{ $t('ssh_keys.copy') }}</b-button>
          <b-button size="sm" variant="danger" @click="removeKey(data.item.comment)">{{ $t('ssh_keys.remove') }}</b-button>
        </template>
      </b-table>
    </div>
  </div>
</template>

<script>
import api from '@/api';

export default {
  name: 'UserSSHKeys',
  props: ['username'],
  data() {
    return {
      authMethod: 'password', // default password-based
      keys: [],
      uploadedKey: null,
      fields: [
        { key: 'comment', label: this.$t('ssh_keys.comment') },
        { key: 'algorithm', label: this.$t('ssh_keys.algorithm') },
        { key: 'fingerprint', label: this.$t('ssh_keys.fingerprint') },
        { key: 'actions', label: this.$t('ssh_keys.actions') },
      ],
    };
  },
  created() {
    this.refresh();
  },
  methods: {
    async refresh() {
      const res = await api.post('user_ssh_list', { user: this.username });
      this.keys = res || [];
    },
    async generateKey() {
      // Uses WebCrypto to generate a key pair client-side
      const keyPair = await window.crypto.subtle.generateKey(
        { name: 'RSA-PSS', modulusLength: 2048, publicExponent: new Uint8Array([1,0,1]), hash: 'SHA-256' },
        true,
        ['sign','verify']
      );
      const exported = await window.crypto.subtle.exportKey('spki', keyPair.publicKey);
      const keyString = btoa(String.fromCharCode(...new Uint8Array(exported)));
      this.keys.push({ key: keyString, comment: 'web-generated', algorithm: 'RSA', fingerprint: keyString.slice(0,8) });
      alert(this.$t('ssh_keys.generated_success'));
    },
    async uploadKey() {
      if (!this.uploadedKey) return;
      const text = await this.uploadedKey.text();
      this.keys.push({ key: text, comment: 'uploaded', algorithm: 'unknown', fingerprint: text.slice(0,8) });
      this.uploadedKey = null;
    },
    copyKey(key) {
      navigator.clipboard.writeText(key);
      alert(this.$t('ssh_keys.copied'));
    },
    async removeKey(comment) {
      this.keys = this.keys.filter(k => k.comment !== comment);
    },
  },
};
</script>

<style scoped>
/* Add basic spacing and alignment */
</style>
