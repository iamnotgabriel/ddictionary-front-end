<template>
  <v-row cols="12" sm="12" md="12">
    <v-col>
      <entry-create-form @created="onCreated" />
    </v-col>
  </v-row>
</template>

<script lang="ts">
import Vue from 'vue'
import { mapGetters } from 'vuex'
import EntryCreateForm from '@/components/Entry/EntryCreateForm.vue'
import { Entry } from '~/dto/Entry'

export default Vue.extend({
  name: 'CreateEntry',
  components: { EntryCreateForm },
  computed: {
    ...mapGetters({ domain: 'domain/domain' }),
  },
  methods: {
    onCreated(entry: Entry) {
      const domain = this.$route.params.domain
      this.$router.push({
        name: 'domains-domain',
        params: {
          domain,
        },
      })
    },
  },
})
</script>

<style></style>
