<template>
  <div>
    <h1 class="title">OAuth2 Client 생성</h1>
    <h2 class="subtitle">OAuth2 Client를 생성합니다.</h2>
    <fade-transition>
      <div v-if="success" class="notification is-success">완료했습니다.</div>
    </fade-transition>
    <form @submit.prevent="updateClient">
      <div class="field">
        <label for="name" class="label">Client ID(client_id)</label>
        <div class="control">
          <input
            id="name"
            v-model="client.id"
            type="text"
            class="input disabled"
            readonly
            disabled
            required
          />
        </div>
      </div>
      <div class="field">
        <label for="name" class="label">Client Secret(client_secret)</label>
        <div class="control">
          <input
            id="name"
            v-model="client.secret"
            type="text"
            class="input disabled"
            required
            readonly
            disabled
          />
        </div>
      </div>
      <div class="field">
        <label for="name" class="label">Client 이름</label>
        <div class="control">
          <input
            id="name"
            v-model="client.name"
            type="text"
            class="input"
            required
          />
        </div>
      </div>
      <div class="field">
        <label for="redirectUris" class="label">Redirect uri들</label>
        <div class="control">
          <textarea
            id="redirectUris"
            v-model="redirectUrisInput"
            required
            class="textarea"
          ></textarea>
        </div>
        <p class="help">개행문자로 구분합니다.</p>
      </div>
      <div class="field">
        <label for="allowedScopes" class="label">허용된 scope들</label>
        <div class="control">
          <input
            id="allowedScopes"
            v-model="allowedScopesInput"
            required
            type="text"
            class="input"
          />
        </div>
        <p class="help">공백 문자로 구분합니다.</p>
      </div>
      <div class="field is-grouped">
        <div class="control">
          <button class="button is-primary" type="submit">수정</button>
          <button class="button is-danger" @click.prevent="renewSecret">
            client_secret 재생성
          </button>
        </div>
      </div>
    </form>
  </div>
</template>

<script lang="ts">
import gql from 'graphql-tag'
import Vue from 'vue'
import hasPermission from '~/middleware/hasPermission'
import FadeTransition from '~/components/fadeTransition.vue'
export default Vue.extend({
  layout: 'admin',
  middleware: hasPermission('oidc.update'),
  components: { FadeTransition },
  data() {
    return {
      client: {
        id: '',
        secret: '',
        name: '',
        allowedScopes: [] as string[],
        redirectUris: [] as string[],
      },
      success: false,
    }
  },
  apollo: {
    client: {
      query: gql`
        query($clientId: ID!) {
          getOAuth2ClientById(id: $clientId) {
            id
            secret
            name
            allowedScopes
            redirectUris
          }
        }
      `,
      variables(): { clientId: string } {
        const { clientId } = this.$route.params
        return {
          clientId,
        }
      },
      update: (data) => data.getOAuth2ClientById,
    },
  },
  computed: {
    allowedScopesInput: {
      get(): string {
        return this.client.allowedScopes.join(' ')
      },
      set(value: string) {
        this.client.allowedScopes = value.split(' ')
      },
    },
    redirectUrisInput: {
      get(): string {
        return this.client.redirectUris.join('\n')
      },
      set(value: string) {
        this.client.redirectUris = value.split('\n')
      },
    },
  },
  methods: {
    async updateClient() {
      this.redirectUrisInput = this.redirectUrisInput.trim()
      this.allowedScopesInput = this.allowedScopesInput.trim()

      await this.$apollo.mutate({
        mutation: gql`
          mutation(
            $id: ID!
            $name: String!
            $redirectUris: [String!]!
            $allowedScopes: [String!]!
          ) {
            updateOAuth2Client(
              id: $id
              name: $name
              redirectUris: $redirectUris
              allowedScopes: $allowedScopes
            ) {
              id
            }
          }
        `,
        variables: {
          id: this.client.id,
          name: this.client.name,
          redirectUris: this.client.redirectUris,
          allowedScopes: this.client.allowedScopes,
        },
      })

      await this.$apollo.queries.client.refetch()
      this.success = true
      setTimeout(() => (this.success = false), 3000)
    },
    async renewSecret() {
      if (!confirm('정말로 client_secret을 재생성하시겠습니까?')) return
      await this.$apollo.mutate({
        mutation: gql`
          mutation($id: ID!) {
            renewOAuth2ClientSecret(id: $id) {
              id
            }
          }
        `,
        variables: {
          id: this.client.id,
        },
      })

      await this.$apollo.queries.client.refetch()
      this.success = true
      setTimeout(() => (this.success = false), 3000)
    },
  },
})
</script>