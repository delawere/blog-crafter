<script setup>
import { ref, computed, watch, onMounted } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { useGetRepos } from '@/queries/get-repos'
import { useGetArticlesQuery } from '@/queries/articles'
import { useGetUser } from '@/queries/get-user'

const { data: user, isLoading: isUserLoading } = useGetUser()
const userLogin = computed(() => user.value?.login)
const enabled = computed(() => !!user.value?.login)
const { data: repos, isFetching, isLoading } = useGetRepos({ user: userLogin }, enabled.value)

const initialValues = {
  pathToContent: localStorage.getItem('pathToContent') || '',
  branch: localStorage.getItem('branch') || 'main'
}
const pathToContent = ref(initialValues.pathToContent)
const branch = ref(initialValues.branch)
const route = useRoute()
const projectId = route.params.projectId

const localState = ref({
  branch: initialValues.branch,
  pathToContent: initialValues.pathToContent
})

const pathToContentValue = computed(() => localState.value?.pathToContent)
const branchValue = computed(() => localState.value?.branch)

const router = useRouter()
const {
  data: articles,
  isFetching: isArticlesFetching,
  isLoading: isArticlesLoading,
  refetch
} = useGetArticlesQuery({
  user: userLogin,
  path: pathToContentValue,
  repo: projectId,
  branch: branchValue
})

onMounted(() => {
  if (localState.value.pathToContent && localState.value.branch) {
    refetch()
  }
})

watch(userLogin, () => {
  if (localState.value.pathToContent && localState.value.branch) {
    refetch()
  }
})

const saveSettings = () => {
  // TODO Save settings to the database
  localStorage.setItem('pathToContent', pathToContent.value)
  localStorage.setItem('branch', branch.value)

  localState.value = {
    pathToContent: pathToContent.value,
    branch: branch.value
  }

  refetch()
}

const repo = computed(() => {
  const filtered = (repos.value || []).filter((t) => t.name === projectId)
  return filtered[0]
})

const isDirty = computed(() => {
  return (
    pathToContent.value !== localState.value.pathToContent ||
    branch.value !== localState.value.branch
  )
})
</script>

<template>
  <div class="mb-6">
    <h3 class="text-h3 mb-4">{{ projectId }}</h3>
    <a :href="repo?.html_url">Open repository</a>
  </div>
  <div class="mb-12">
    <h3 class="text-h4 mb-4">Settings</h3>
    <v-text-field v-model="pathToContent" label="Path to content" />
    <v-text-field v-model="branch" label="Branch name (Optional)" />
    <v-btn
      variant="tonal"
      color="#5865f2"
      class="text-subtitle-1"
      @click="saveSettings"
      :disabled="!isDirty"
    >
      <v-icon icon="check" />
      Save settings</v-btn
    >
  </div>

  <h3 class="text-h4 mb-4">Articles</h3>
  <div>
    <v-btn variant="tonal" color="#5865f2" class="text-subtitle-1"
      ><router-link :to="`/projects/${projectId}/articles/new`">New Article</router-link></v-btn
    >
  </div>
  <div>
    <v-skeleton-loader
      width="500"
      v-if="isArticlesFetching || isArticlesFetching"
      type="list-item-three-line"
    ></v-skeleton-loader>
    <v-list v-else>
      <v-list-item v-for="{ parent } in articles" :key="parent" class="pl-0">
        <router-link :to="`/projects/${projectId}/articles/${encodeURIComponent(parent)}`">
          {{ parent.split('/').at(-1) }}</router-link
        >
      </v-list-item>
    </v-list>
  </div>
</template>