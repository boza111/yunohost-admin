<template>
  <div>
    <h2>{{ $t('tools_ssh.title') }}</h2>

    <!-- Toggle root account enable/disable -->
    <b-form-group label="{{ $t('tools_ssh.root_account') }}">
      <b-form-switch v-model="rootEnabled" @change="toggleRoot">
        {{ rootEnabled ? $t('tools_ssh.enabled') : $t('tools_ssh.disabled') }}
      </b-form-switch>
      <b-alert variant="info">{{ $t('tools_ssh.root_info') }}</b-alert>
    </b-form-group>

    <!-- Toggle authentication method for root -->
    <b-form-group label="{{ $t('tools_ssh.auth_method') }}">
      <b-form-radio-group v-model="authMethod" @change="changeAuthMethod">
        <b-form-radio value="password">{{ $t('tools_ssh.password') }}</b-form-radio>
        <b-form-radio value="keys">{{ $t('tools_ssh.ssh_key') }}</b-form-radio>
      </b-form-radio-group>
      <b-alert variant="warning" v-if="authMethod === 'keys'">
        {{ $t('tools_ssh.warning') }}
      </b-alert>
    </b-form-group>
  </div>
</template>

<script>
import api from '@/api';

export default {
  name: 'ToolsSSHSettings',
  data() {
    return {
      rootEnabled: false,
      authMethod: 'password',
    };
  },
  created() {
    this.refresh();
  },
  methods: {
    async refresh() {
      const res = await api.post('ssh_root_status');
      this.rootEnabled = res.enabled;
      this.authMethod = res.authMethod;
    },
    async toggleRoot() {
      await api.post('ssh_toggle_root', { enabled: this.rootEnabled });
    },
    async changeAuthMethod() {
      await api.post('ssh_set_auth', { auth: this.authMethod });
    },
  },
};
</script>

<style scoped>
/* spacing for toggles */
</style>
