<script context="module">
  export const prerender = true
</script>

<script>
  import { onDestroy, setContext } from 'svelte'
  import { browser } from '$app/environment'
  import Primo, {
    modal as primoModal,
    PrimoFieldTypes,
    fieldTypes,
    stores,
  } from '@primo-app/primo'
  import modal from '@primo-app/primo/src/stores/app/modal'
  import Build from '../../extensions/Build.svelte'
  import ImageField from '../../extensions/FieldTypes/ImageField.svelte'
  import * as actions from '../../actions'
  import user from '../../stores/user'
  import { sitePassword } from '../../stores/misc'
  import { activeSite } from '../../stores/site'
  import { page } from '$app/stores'
  import * as primo from '@primo-app/primo/package.json'
  import config from '../../stores/config'
  import LockAlert from '$lib/components/LockAlert.svelte'

  $: siteID = $page.params.site

  if (browser) {
    primoModal.register([
      {
        id: 'BUILD',
        component: Build,
        componentProps: {
          siteName: 'Website', // TODO - change
        },
        options: {
          route: 'build',
          width: 'md',
          header: {
            title: 'Build to Github',
            icon: 'fab fa-github',
          },
          hideLocaleSelector: true,
        },
      },
    ])
    fieldTypes.register([
      {
        id: 'image',
        label: 'Image',
        component: ImageField,
      },
    ])
  }

  let currentPath
  let siteLocked = false
  async function fetchSite(fullPath) {
    if (currentPath === fullPath) return
    currentPath = fullPath
    const res = await actions.sites.get(siteID)
    if (res?.active_editor && res.active_editor !== $user.email) {
      siteLocked = true
      modal.show('DIALOG', {
        component: LockAlert,
        componentProps: {
          email: res.active_editor,
          canGoToDashboard: false,
        },
        options: {
          disableClose: true,
        },
      })
    } else if (res) {
      actions.setActiveEditor({ siteID })
      $activeSite = res
    }
  }

  async function saveData(updatedSite) {
    saving = true
    const success = await actions.sites.save(updatedSite)
    stores.saved.set(success)
    saving = false
  }

  let saving = false

  $: if ($user.signedIn && browser) fetchSite($page.url.pathname)

  $: if (browser && $sitePassword) {
    setContext('hidePrimoButton', true)
  }

  onDestroy(() => {
    if (browser && !siteLocked) actions.setActiveEditor({ siteID, lock: false })
    else if (siteLocked) modal.hide()
  })
</script>

<Primo
  data={$activeSite}
  role={$user.role}
  options={{
    logo: $config.customization.logo.url,
  }}
  {saving}
  on:save={async ({ detail: data }) => saveData(data)}
/>
<slot />
<div id="app-version">
  <span>primo v{primo.version}</span>
  <span>server v{__SERVER_VERSION__}</span>
</div>

<style global lang="postcss">
  .primo-reset {
    @tailwind base;
    font-family: 'Satoshi', sans-serif !important;

    button,
    button * {
      cursor: pointer;
    }
  }

  body {
    margin: 0;
  }

  #app-version {
    font-family: 'Satoshi', sans-serif;
    font-size: 0.75rem;
    color: var(--color-gray-4);
    position: fixed;
    bottom: 0.5rem;
    left: 0.5rem;

    span:first-child {
      margin-right: 0.5rem;
    }
  }
</style>
