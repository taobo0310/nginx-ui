<script setup lang="ts">
import type { RuntimeInfo } from '@/api/upgrade'
import type { Ref } from 'vue'
import upgrade from '@/api/upgrade'

import websocket from '@/lib/websocket'
import version from '@/version.json'
import dayjs from 'dayjs'
import { marked } from 'marked'
import { useRoute } from 'vue-router'

const route = useRoute()
const data = ref({}) as Ref<RuntimeInfo>
const lastCheck = ref('')
const loading = ref(false)
const channel = ref('stable')

const progressStrokeColor = {
  from: '#108ee9',
  to: '#87d068',
}

const modalVisible = ref(false)
const progressPercent = ref(0)
const progressStatus = ref('active') as Ref<'normal' | 'active' | 'success' | 'exception'>
const modalClosable = ref(false)
const getReleaseError = ref(false)

const progressPercentComputed = computed(() => {
  return Number.parseFloat(progressPercent.value.toFixed(1))
})

function getLatestRelease() {
  loading.value = true
  data.value.body = ''
  upgrade.get_latest_release(channel.value).then(r => {
    data.value = r
    lastCheck.value = dayjs().format('YYYY-MM-DD HH:mm:ss')
  }).catch(e => {
    getReleaseError.value = e?.message
  }).finally(() => {
    loading.value = false
  })
}

getLatestRelease()

watch(channel, getLatestRelease)

const isLatestVer = computed(() => {
  return data.value.name === `v${version.version}`
})

const logContainer = useTemplateRef('logContainer')

function log(msg: string) {
  const para = document.createElement('p')

  para.appendChild(document.createTextNode($gettext(msg)))

  logContainer.value!.appendChild(para)

  logContainer.value!.scroll({ top: 320, left: 0, behavior: 'smooth' })
}

const dryRun = computed(() => {
  return !!route.query.dry_run
})

async function performUpgrade() {
  progressStatus.value = 'active'
  modalClosable.value = false
  modalVisible.value = true
  progressPercent.value = 0
  logContainer.value!.innerHTML = ''

  log($gettext('Upgrading Nginx UI, please wait...'))

  const ws = websocket('/api/upgrade/perform', false)

  let last = 0

  ws.onopen = () => {
    ws.send(JSON.stringify({
      dry_run: dryRun.value,
      channel: channel.value,
    }))
  }

  let isFailed = false

  ws.onmessage = async m => {
    const r = JSON.parse(m.data)
    if (r.message)
      log(r.message)
    switch (r.status) {
      case 'info':
        progressPercent.value += 10
        break
      case 'progress':
        progressPercent.value += (r.progress - last) / 2
        last = r.progress
        break
      case 'error':
        progressStatus.value = 'exception'
        modalClosable.value = true
        isFailed = true
        break
      default:
        modalClosable.value = true
        break
    }
  }

  ws.onerror = () => {
    isFailed = true
    progressStatus.value = 'exception'
    modalClosable.value = true
  }

  ws.onclose = async () => {
    if (isFailed)
      return

    const t = setInterval(() => {
      upgrade.current_version().then(() => {
        clearInterval(t)
        progressStatus.value = 'success'
        progressPercent.value = 100
        modalClosable.value = true
        log('Upgraded successfully')

        setInterval(() => {
          location.reload()
        }, 1000)
      })
    }, 2000)
  }
}
</script>

<template>
  <ACard :title="$gettext('Upgrade')">
    <AModal
      v-model:open="modalVisible"
      :title="$gettext('Core Upgrade')"
      :mask-closable="false"
      :footer="null"
      :closable="modalClosable"
      force-render
    >
      <AProgress
        :stroke-color="progressStrokeColor"
        :percent="progressPercentComputed"
        :status="progressStatus"
      />

      <div
        ref="logContainer"
        class="core-upgrade-log-container"
      />
    </AModal>
    <div class="upgrade-container">
      <p>{{ $gettext('You can check Nginx UI upgrade at this page.') }}</p>
      <h3>{{ $gettext('Current Version') }}: v{{ version.version }}</h3>
      <template v-if="getReleaseError">
        <AAlert
          type="error"
          :title="$gettext('Get release information error')"
          :message="getReleaseError"
          banner
        />
      </template>
      <template v-else>
        <p>{{ $gettext('OS') }}: {{ data.os }}</p>
        <p>{{ $gettext('Arch') }}: {{ data.arch }}</p>
        <p>{{ $gettext('Executable Path') }}: {{ data.ex_path }}</p>
        <p>
          {{ $gettext('Last checked at') }}: {{ lastCheck }}
          <AButton
            type="link"
            :loading="loading"
            @click="getLatestRelease"
          >
            {{ $gettext('Check again') }}
          </AButton>
        </p>
        <AFormItem :label="$gettext('Channel')">
          <ASelect v-model:value="channel">
            <ASelectOption key="stable">
              {{ $gettext('Stable') }}
            </ASelectOption>
            <ASelectOption key="prerelease">
              {{ $gettext('Pre-release') }}
            </ASelectOption>
          </ASelect>
        </AFormItem>
        <template v-if="!loading">
          <AAlert
            v-if="isLatestVer"
            type="success"
            :message="$gettext('You are using the latest version')"
            banner
          />
          <AAlert
            v-else
            type="info"
            :message="$gettext('New version released')"
            banner
          />
          <template v-if="dryRun">
            <br>
            <AAlert
              type="info"
              :message="$gettext('Dry run mode enabled')"
              banner
            />
          </template>
          <div class="control-btn">
            <ASpace>
              <AButton
                type="primary"
                ghost
                @click="performUpgrade"
              >
                {{ isLatestVer ? $gettext('Reinstall') : $gettext('Upgrade') }}
              </AButton>
            </ASpace>
          </div>
        </template>
      </template>
      <template v-if="data.body">
        <h2 class="latest-version">
          {{ data.name }}
          <ATag
            v-if="channel === 'stable'"
            color="green"
          >
            {{ $gettext('Stable') }}
          </ATag>
          <ATag
            v-if="channel === 'prerelease'"
            color="blue"
          >
            {{ $gettext('Pre-release') }}
          </ATag>
        </h2>

        <h3>{{ $gettext('Release Note') }}</h3>
        <div v-dompurify-html="marked.parse(data.body)" />
      </template>
    </div>
  </ACard>
</template>

<style lang="less">
.dark {
  .core-upgrade-log-container {
    background-color: rgba(0, 0, 0, 0.84);
  }
}

.core-upgrade-log-container {
  height: 320px;
  overflow: scroll;
  background-color: #f3f3f3;
  border-radius: 4px;
  margin-top: 15px;
  padding: 10px;

  p {
    font-size: 12px;
    line-height: 1.3;
  }
}
</style>

<style lang="less" scoped>
.upgrade-container {
  width: 100%;
  max-width: 600px;
  margin: 0 auto;
  padding: 0 10px;
}

.control-btn {
  margin: 15px 0;
}

.latest-version {
  display: flex;
  align-items: center;

  span.ant-tag {
    margin-left: 10px;
  }
}
</style>
