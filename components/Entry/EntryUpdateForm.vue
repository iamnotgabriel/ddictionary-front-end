<template>
  <v-card>
    <v-card-title>Update Entry</v-card-title>
    <v-card-text>
      <v-row>
        <v-col>
          <v-form class="form">
            <!-- title -->
            <v-text-field v-model="title" required label="Title">
            </v-text-field>
            <!-- group -->
            <v-combobox
              v-model="group"
              :items="groups"
              label="Group"
              chips
            ></v-combobox>
            <!-- definitions -->
            <v-list>
              <v-subheader>Definitions</v-subheader>
              <v-list-item-group>
                <draggable v-model="definitions">
                  <transition-group>
                    <v-list-item
                      v-for="(def, i) of definitions"
                      :key="def.order"
                      @mouseover="def.showEdit = true"
                      @mouseleave="def.showEdit = false"
                    >
                      <v-list-item-content>
                        <vue-markdown> {{ def.text }} </vue-markdown>
                      </v-list-item-content>
                      <v-list-item-action v-show="def.showEdit">
                        <span>
                          <v-icon @click="removeDefinitions(i)">
                            mdi-delete
                          </v-icon>
                          <v-icon @click="updateDefinition(i)">
                            mdi-lead-pencil
                          </v-icon>
                        </span>
                      </v-list-item-action>
                    </v-list-item>
                  </transition-group>
                </draggable>
              </v-list-item-group>
            </v-list>
            <!-- definition input -->
            <client-only placeholder="Loading Your Editor...">
              <vue-simplemde
                ref="markdownEditor"
                v-model="currentDefinitionText"
                :configs="editor"
                placeholder="Write your definition"
                @input="addDefinition()"
              ></vue-simplemde>
              <v-btn class="text--button" @click="addDefinition">
                add definition
              </v-btn>
            </client-only>
            <!-- translations -->
            <v-list v-if="translations.length">
              <v-subheader>Translations</v-subheader>
              <v-list-item-group>
                <v-list-item v-for="(trans, i) of translations" :key="i">
                  <v-list-item-content>
                    {{ trans }}
                  </v-list-item-content>
                  <v-list-item-action>
                    <v-icon @click="deleteTranslation(i)">mdi-close</v-icon>
                  </v-list-item-action>
                </v-list-item>
              </v-list-item-group>
            </v-list>
            <!-- translation input -->
            <v-text-field
              v-model="translation"
              counter="25"
              label="Translation"
              @keyup.enter="addTransaction()"
            >
            </v-text-field>
            <!-- image -->
            <v-file-input
              :rules="imageRules"
              accept="image/png, image/jpeg, image/bmp"
              label="Image"
              placeholder="Pick an image"
              prepend-icon="mdi-image"
              show-size
              @change="addImage"
            ></v-file-input>
          </v-form>
        </v-col>
      </v-row>
    </v-card-text>
    <v-card-actions>
      <v-btn color="primary" @click="submit">create</v-btn>
      <v-btn outlined color="normal" @click="emitCancel">cancel</v-btn>
    </v-card-actions>
  </v-card>
</template>

<script lang="ts">
import Vue from 'vue'
import draggable from 'vuedraggable'
import VueMarkdown from 'vue-markdown'
import { mapGetters } from 'vuex'
import { EntryDefinition } from './Definition'

export default Vue.extend({
  name: 'EntryUpdateForm',
  components: {
    draggable,
    VueMarkdown,
  },
  props: {
    id: {
      type: String,
      required: true,
    },
  },
  data() {
    return {
      imageRules: [
        (file: File) =>
          !file || file.size < 2_000_000 || 'Image should be less than 2MB!',
      ],
      title: '',
      group: '',
      groups: [] as string[],
      currentDefinitionText: '',
      definitionCount: 0,
      translation: '',
      image: undefined as unknown as string,
      definitions: [] as EntryDefinition[],
      translations: [] as string[],
      editor: {
        toolbar: [
          'bold',
          'italic',
          'heading',
          '|',
          'heading-smaller',
          'heading-bigger',
          'quote',
          '|',
          'table',
          'link',
          'code',
          'guide',
          'clean-block',
        ],
        spellChecker: false,
      },
    }
  },
  head() {
    return {
      title: 'create entry',
    }
  },
  computed: {
    ...mapGetters({ domain: 'domain/domain' }),
  },
  async mounted() {
    const entry = await this.$axios.$get(`/entries/${this.$props.id}`).catch()
    this.title = entry.title
    this.group = entry.group
    this.definitions = new Array(entry.definitions.length)
    this.image = entry.image
    this.translations = entry.translations

    entry.definitions.forEach((text: string, order: number) => {
      this.definitions[order] = { text, order, showEdit: false }
    })
    const groups = await this.$axios.$get(`/domains/${this.domain.slug}/groups`)
    this.groups = groups.map((g: { slug: string }) => g.slug)
  },
  methods: {
    emitCancel() {
      this.$emit('canceled')
    },
    isValid() {
      if (!this.title) {
        return false
      }
      if (!this.group) {
        return false
      }
      if (this.definitions.length === 0) {
        return false
      }
      return true
    },
    async submit() {
      if (!this.isValid()) {
        return
      }
      await this.addGroup()
      await this.update()
    },
    getFormData(options: any = {}) {
      const data = {
        domain: this.domain.slug,
        title: this.title,
        group: this.group,
        definitions: this.definitions.map((def) => def.text),
        translations: this.translations,
        image: this.image,
      }
      Object.apply(data, options)
      return data
    },
    async update() {
      const data = this.getFormData({ domain: undefined })
      await this.$axios.$patch(`/entries/${this.$props.id}`, data)
      this.$emit('updated', data)
    },
    async addGroup() {
      if (!this.groups.includes(this.group)) {
        const domainSlug = this.domain.slug
        await this.$axios
          .$post(`/domains/${domainSlug}/groups`, {
            slug: this.group,
          })
          .catch((e) => console.error(e))
      }
    },
    addDefinition() {
      if (this.currentDefinitionText) {
        this.definitions.push({
          text: this.currentDefinitionText,
          showEdit: false,
          order: this.definitionCount++,
        })
        this.currentDefinitionText = ''
        const simplemde = (this.$refs.markdownEditor as any).simplemde
        simplemde.value(this.currentDefinitionText)
      }
    },
    removeDefinitions(i: number) {
      this.definitions.splice(i, 1)
    },
    updateDefinition(i: number) {
      this.currentDefinitionText = this.definitions[i].text
      this.removeDefinitions(i)
    },
    addTransaction() {
      if (this.translation.length <= 25) {
        this.translations.push(this.translation)
        this.translation = ''
      }
    },
    addImage(file: File) {
      if (file) {
        const reader = new FileReader()
        reader.onloadend = () => {
          this.image = reader.result as string
        }
        reader.readAsDataURL(file)
      }
    },
    deleteTranslation(i: number) {
      this.translations.splice(i, 1)
    },
  },
})
</script>

<style>
.markdown-editor .CodeMirror {
  height: 150px;
}

.markdown-editor .CodeMirror,
.markdown-editor .CodeMirror-scroll {
  min-height: 150px;
}
.form {
  margin: 0px 16px 0px 16px;
}
</style>
