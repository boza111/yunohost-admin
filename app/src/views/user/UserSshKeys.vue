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
      <b-button variant="primary" class="mb-2" @click="generateKey">
        {{ $t('ssh_keys.generate') }}
      </b-button>
      <b-form-file
        v-model="uploadedKey"
        :placeholder="$t('ssh_keys.upload_placeholder')"
        class="mb-2"
      ></b-form-file>
      <b-button variant="success" class="mb-2" @click="uploadKey">
        {{ $t('ssh_keys.upload') }}
      </b-button>

      <b-table striped hover :items="keys" :fields="fields" small>
        <template #cell(actions)="data">
          <b-button size="sm" variant="info" @click="copyKey(data.item.key)">
            {{ $t('ssh_keys.copy') }}
          </b-button>
          <b-button
            size="sm"
            variant="danger"
            @click="removeKey(data.item.comment)"
          >
            {{ $t('ssh_keys.remove') }}
          </b-button>
        </template>
      </b-table>
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref, onMounted } from 'vue';
import api from '@/api';

export default defineComponent({
  name: 'UserSSHKeys',
  props: {
    username: {
      type: String,
      required: true,
    },
  },
  setup(props) {
    const authMethod = ref<'password' | 'keys'>('password');
    const keys = ref<Array<{ key: string; comment: string; algorithm: string; fingerprint: string }>>([]);
    const uploadedKey = ref<File | null>(null);

    const fields = [
      { key: 'comment', label: 'Comment', sortable: true },
      { key: 'algorithm', label: 'Algorithm', sortable: true },
      { key: 'fingerprint', label: 'Fingerprint', sortable: true },
      { key: 'actions', label: 'Actions' },
    ];

    async function refresh() {
      try {
        const res = await api.post('get_user_ssh_keys', { username: props.username });
        keys.value = res || [];
      } catch (err) {
        console.error('Failed to load SSH keys', err);
      }
    }

    async function generateKey() {
      try {
        const keyPair = await window.crypto.subtle.generateKey(
          { name: 'RSA-PSS', modulusLength: 2048, publicExponent: new Uint8Array([1, 0, 1]), hash: 'SHA-256' },
          true,
          ['sign', 'verify']
        );
        const exported = await window.crypto.subtle.exportKey('spki', keyPair.publicKey);
        const keyString = btoa(String.fromCharCode(...new Uint8Array(exported)));

        await api.post('install_user_ssh_key', {
          username: props.username,
          key_data: keyString,
          comment: 'web-generated',
        });

        await refresh();
        alert('SSH key generated and installed successfully');
      } catch (err) {
        console.error('Failed to generate SSH key', err);
        alert('Error generating SSH key');
      }
    }

    async function uploadKey() {
      if (!uploadedKey.value) return;

      try {
        const text = await uploadedKey.value.text();
        if (!text.startsWith('ssh-')) {
          alert('Invalid SSH key format');
          return;
        }

        await api.post('install_user_ssh_key', {
          username: props.username,
          key_data: text,
          comment: 'uploaded',
        });

        uploadedKey.value = null;
        await refresh();
        alert('SSH key uploaded successfully');
      } catch (err) {
        console.error('Failed to upload SSH key', err);
        alert('Error uploading SSH key');
      }
    }

    async function removeKey(comment: string) {
      try {
        await api.post('remove_user_ssh_key', { username: props.username, comment });
        await refresh();
      } catch (err) {
        console.error('Failed to remove SSH key', err);
        alert('Error removing SSH key');
      }
    }

    function copyKey(key: string) {
      navigator.clipboard.writeText(key).then(() => {
        alert('SSH key copied to clipboard');
      });
    }

    onMounted(refresh);

    return { authMethod, keys, uploadedKey, fields, generateKey, uploadKey, removeKey, copyKey };
  },
});
</script>

<style scoped>
.mb-2 {
  margin-bottom: 0.5rem;
}
</style>
