<script>
  import { onMount } from 'svelte'

  const storageKey = 'method-catatan-items'
  const themeKey = 'method-catatan-theme'

  const createEmptyForm = () => ({
    programmingLanguage: '',
    dataType: '',
    methodName: '',
    description: '',
    usage: '',
  })

  const initialItems = [
    {
      id: crypto.randomUUID(),
      programmingLanguage: 'JavaScript',
      dataType: 'Array',
      methodName: 'Array.map()',
      description: 'Creates a new array by transforming every item from the source array.',
      usage:
        'Use it when you need to return a transformed version of an array without mutating the original data.',
      createdAt: new Date().toISOString(),
    },
  ]

  const loadItems = () => {
    if (typeof window === 'undefined') {
      return initialItems
    }

    const savedItems = window.localStorage.getItem(storageKey)

    if (!savedItems) {
      window.localStorage.setItem(storageKey, JSON.stringify(initialItems))
      return initialItems
    }

    try {
      const parsedItems = JSON.parse(savedItems)
      return Array.isArray(parsedItems) ? parsedItems.map(normalizeItem) : initialItems
    } catch (error) {
      window.localStorage.setItem(storageKey, JSON.stringify(initialItems))
      return initialItems
    }
  }

  const normalizeItem = (item) => ({
    ...item,
    programmingLanguage: item.programmingLanguage || 'JavaScript',
  })

  const getPreferredTheme = () => {
    if (typeof window === 'undefined') {
      return 'light'
    }

    const savedTheme = window.localStorage.getItem(themeKey)

    if (savedTheme === 'light' || savedTheme === 'dark') {
      return savedTheme
    }

    return window.matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : 'light'
  }

  let items = loadItems()
  let form = createEmptyForm()
  let editingId = null
  let isFormModalOpen = false
  let deleteTargetId = null
  let detailItemId = null
  let tableItemId = null
  let detailItem = null
  let tableItem = null
  let theme = getPreferredTheme()
  let alertState = {
    status: 'info',
    title: '',
    description: '',
  }

  onMount(() => {
    applyTheme(theme)
  })

  $: detailItem = items.find((item) => item.id === detailItemId) ?? null
  $: tableItem = items.find((item) => item.id === tableItemId) ?? null

  const syncItems = (nextItems) => {
    items = nextItems.map(normalizeItem)

    if (typeof window !== 'undefined') {
      window.localStorage.setItem(storageKey, JSON.stringify(items))
    }
  }

  const showAlert = (status, title, description) => {
    alertState = { status, title, description }
  }

  const applyTheme = (nextTheme) => {
    theme = nextTheme

    if (typeof document !== 'undefined') {
      document.documentElement.dataset.theme = nextTheme
    }

    if (typeof window !== 'undefined') {
      window.localStorage.setItem(themeKey, nextTheme)
    }
  }

  const toggleTheme = () => {
    applyTheme(theme === 'light' ? 'dark' : 'light')
  }

  const clearAlert = () => {
    alertState = {
      status: 'info',
      title: '',
      description: '',
    }
  }

  const resetForm = () => {
    form = createEmptyForm()
    editingId = null
  }

  const openCreateModal = () => {
    resetForm()
    clearAlert()
    isFormModalOpen = true
  }

  const openEditModal = (item) => {
    editingId = item.id
    form = {
      programmingLanguage: item.programmingLanguage,
      dataType: item.dataType,
      methodName: item.methodName,
      description: item.description,
      usage: item.usage,
    }
    clearAlert()
    isFormModalOpen = true
  }

  const closeFormModal = () => {
    isFormModalOpen = false
    resetForm()
  }

  const submitForm = () => {
    const payload = {
      programmingLanguage: form.programmingLanguage.trim(),
      dataType: form.dataType.trim(),
      methodName: form.methodName.trim(),
      description: form.description.trim(),
      usage: form.usage.trim(),
    }

    if (
      !payload.programmingLanguage ||
      !payload.dataType ||
      !payload.methodName ||
      !payload.description ||
      !payload.usage
    ) {
      showAlert('error', 'Incomplete form', 'Please fill in every field before saving.')
      return
    }

    if (editingId) {
      syncItems(
        items.map((item) =>
          item.id === editingId
            ? {
                ...item,
                ...payload,
              }
            : item,
        ),
      )

      showAlert('success', 'Method updated', `"${payload.methodName}" was updated successfully.`)
    } else {
      syncItems([
        {
          id: crypto.randomUUID(),
          ...payload,
          createdAt: new Date().toISOString(),
        },
        ...items,
      ])

      showAlert('success', 'Method saved', `"${payload.methodName}" was added to your collection.`)
    }

    closeFormModal()
  }

  const openDetailModal = (item) => {
    detailItemId = item.id
  }

  const closeDetailModal = () => {
    detailItemId = null
  }

  const openTableModal = (item) => {
    tableItemId = item.id
  }

  const closeTableModal = () => {
    tableItemId = null
  }

  const openEditFromDetail = (item) => {
    closeDetailModal()
    openEditModal(item)
  }

  const askDeleteFromDetail = (id) => {
    closeDetailModal()
    askDelete(id)
  }

  const askDelete = (id) => {
    deleteTargetId = id
    clearAlert()
  }

  const closeDeleteModal = () => {
    deleteTargetId = null
  }

  const confirmDelete = () => {
    const targetItem = items.find((item) => item.id === deleteTargetId)

    if (!targetItem) {
      closeDeleteModal()
      return
    }

    syncItems(items.filter((item) => item.id !== deleteTargetId))
    showAlert('success', 'Method deleted', `"${targetItem.methodName}" was removed successfully.`)
    closeDeleteModal()
    closeDetailModal()
    closeTableModal()

    if (editingId === deleteTargetId) {
      closeFormModal()
    }
  }

  const exportToPdf = () => {
    if (!items.length) {
      showAlert('warning', 'Nothing to export', 'Add at least one method before exporting to PDF.')
      return
    }

    const printWindow = window.open('', '_blank', 'width=960,height=720')

    if (!printWindow) {
      showAlert(
        'error',
        'Popup blocked',
        'Please allow popups in your browser and try exporting again.',
      )
      return
    }

    const exportedAt = new Date().toLocaleString('en-US', {
      dateStyle: 'full',
      timeStyle: 'short',
    })

    const rows = items
      .map(
        (item, index) => `
          <tr>
            <td>${index + 1}</td>
            <td>${escapeHtml(item.programmingLanguage)}</td>
            <td>${escapeHtml(item.dataType)}</td>
            <td>${escapeHtml(item.methodName)}</td>
            <td>${escapeHtml(item.description).replace(/\n/g, '<br>')}</td>
            <td>${escapeHtml(item.usage).replace(/\n/g, '<br>')}</td>
          </tr>
        `,
      )
      .join('')

    printWindow.document.write(`
      <!doctype html>
      <html lang="en">
        <head>
          <meta charset="UTF-8" />
          <title>Method Notes Export</title>
          <style>
            * {
              box-sizing: border-box;
            }

            body {
              margin: 0;
              padding: 32px;
              font-family: Arial, Helvetica, sans-serif;
              color: #000000;
              background: #ffffff;
            }

            .print-shell {
              max-width: 1120px;
              margin: 0 auto;
            }

            h1 {
              margin: 0 0 8px;
              font-size: 28px;
            }

            p {
              margin: 0 0 6px;
              font-size: 14px;
            }

            table {
              width: 100%;
              border-collapse: collapse;
              margin-top: 24px;
            }

            th,
            td {
              border: 1px solid #000000;
              padding: 10px;
              text-align: left;
              vertical-align: top;
              font-size: 13px;
              line-height: 1.5;
            }

            th {
              background: #ffffff;
              font-weight: 700;
            }

            @media print {
              body {
                padding: 0;
              }
            }
          </style>
        </head>
        <body>
          <main class="print-shell">
            <h1>Method Notes</h1>
            <p>Exported at: ${escapeHtml(exportedAt)}</p>
            <p>Total methods: ${items.length}</p>

            <table>
              <thead>
                <tr>
                  <th>No.</th>
                  <th>Programming Language</th>
                  <th>Data Type</th>
                  <th>Method Name</th>
                  <th>Description</th>
                  <th>Usage</th>
                </tr>
              </thead>
              <tbody>${rows}</tbody>
            </table>
          </main>
          <script>
            window.onload = () => {
              window.print();
            };
          <\/script>
        </body>
      </html>
    `)
    printWindow.document.close()
    showAlert('success', 'PDF ready', 'The printable PDF view has been opened in a new window.')
  }

  const formatDate = (value) =>
    new Date(value).toLocaleString('en-US', {
      dateStyle: 'medium',
      timeStyle: 'short',
    })

  const getDeleteTarget = () => items.find((item) => item.id === deleteTargetId)

  const handleCardKeydown = (event, item) => {
    if (event.key === 'Enter' || event.key === ' ') {
      event.preventDefault()
      openDetailModal(item)
    }
  }

  const escapeHtml = (value) =>
    value
      .replaceAll('&', '&amp;')
      .replaceAll('<', '&lt;')
      .replaceAll('>', '&gt;')
      .replaceAll('"', '&quot;')
      .replaceAll("'", '&#39;')
</script>

<svelte:head>
  <title>Method Notes</title>
  <meta
    name="description"
    content="A simple CRUD app for saving method names, descriptions, and usage notes."
  />
</svelte:head>

<main class="page-shell">
  <section class="hero-card">
    <div class="hero-copy">
      <p class="eyebrow">Method Manager</p>
      <h1>Save your favorite methods in clean cards and export them as a plain PDF table.</h1>
      <p class="lead">
        Everything is stored in your browser, so you can quickly manage method references without
        setting up a database.
      </p>
    </div>

    <div class="hero-actions">
      <button class="theme-button" type="button" on:click={toggleTheme}>
        {theme === 'light' ? 'Dark Mode' : 'Light Mode'}
      </button>
      <button class="primary-button" type="button" on:click={openCreateModal}>Add Method</button>
      <button class="secondary-button" type="button" on:click={exportToPdf}>Export PDF</button>
    </div>
  </section>

  {#if alertState.title}
    <section class={`chakra-alert chakra-alert--${alertState.status}`} role="alert" aria-live="polite">
      <div class="chakra-alert__icon" aria-hidden="true">
        {#if alertState.status === 'success'}
          ✓
        {:else if alertState.status === 'error'}
          !
        {:else if alertState.status === 'warning'}
          !
        {:else}
          i
        {/if}
      </div>

      <div class="chakra-alert__content">
        <strong>{alertState.title}</strong>
        <p>{alertState.description}</p>
      </div>

      <button class="alert-close" type="button" on:click={clearAlert} aria-label="Close alert">
        ×
      </button>
    </section>
  {/if}

  <section class="stats-grid">
    <article class="stats-card">
      <span>Total Methods</span>
      <strong>{items.length}</strong>
    </article>
    <article class="stats-card">
      <span>Current View</span>
      <strong>{isFormModalOpen ? (editingId ? 'Editing' : 'Creating') : 'Browsing'}</strong>
    </article>
    <article class="stats-card">
      <span>Theme</span>
      <strong>{theme === 'light' ? 'Light' : 'Dark'}</strong>
    </article>
    <article class="stats-card">
      <span>Languages</span>
      <strong>{new Set(items.map((item) => item.programmingLanguage)).size}</strong>
    </article>
  </section>

  {#if items.length}
    <section class="cards-grid">
      {#each items as item}
        <article class="method-card">
          <div
            class="card-surface"
            role="button"
            tabindex="0"
            aria-label={`Open details for ${item.methodName}`}
            on:click={() => openDetailModal(item)}
            on:keydown={(event) => handleCardKeydown(event, item)}
          >
            <div class="card-top">
              <div>
                <p class="card-label">Programming Language</p>
                <h2>{item.programmingLanguage}</h2>
              </div>
              <span class="timestamp">{formatDate(item.createdAt)}</span>
            </div>

            <div class="card-section">
              <p class="card-label">Method Name</p>
              <h3>{item.methodName}</h3>
            </div>

            <div class="card-section card-section--split">
              <div>
                <p class="card-label">Data Type</p>
                <p class="pill-text">{item.dataType}</p>
              </div>
              <div>
                <p class="card-label">Description</p>
                <p>{item.description}</p>
              </div>
            </div>
          </div>

          <div class="card-actions">
            <button class="secondary-button" type="button" on:click={() => openDetailModal(item)}>View</button>
            <button class="ghost-button" type="button" on:click={() => openEditModal(item)}>Edit</button>
            <button class="danger-button" type="button" on:click={() => askDelete(item.id)}>Delete</button>
          </div>
        </article>
      {/each}
    </section>
  {:else}
    <section class="empty-card">
      <h2>No methods saved yet</h2>
      <p>Open the popup form to create your first method note.</p>
      <button class="primary-button" type="button" on:click={openCreateModal}>Create First Method</button>
    </section>
  {/if}

  {#if isFormModalOpen}
    <div class="modal-backdrop" role="presentation" on:click={closeFormModal}>
      <div
        class="modal-card"
        role="dialog"
        aria-modal="true"
        aria-labelledby="method-modal-title"
        on:click|stopPropagation
        on:keydown|stopPropagation
        tabindex="-1"
      >
        <div class="modal-header">
          <div>
            <p class="eyebrow">{editingId ? 'Edit Method' : 'New Method'}</p>
            <h2 id="method-modal-title">{editingId ? 'Update method details' : 'Create a new method note'}</h2>
          </div>
          <button class="icon-button" type="button" on:click={closeFormModal} aria-label="Close form">
            ×
          </button>
        </div>

        <form class="modal-form" on:submit|preventDefault={submitForm}>
          <label class="field">
            <span>Programming Language</span>
            <input
              bind:value={form.programmingLanguage}
              type="text"
              name="programmingLanguage"
              placeholder="Example: JavaScript"
            />
          </label>

          <label class="field">
            <span>Data Type</span>
            <input bind:value={form.dataType} type="text" name="dataType" placeholder="Example: String" />
          </label>

          <label class="field">
            <span>Method Name</span>
            <input
              bind:value={form.methodName}
              type="text"
              name="methodName"
              placeholder="Example: String.includes()"
            />
          </label>

          <label class="field">
            <span>Description</span>
            <textarea
              bind:value={form.description}
              name="description"
              rows="4"
              placeholder="Write a short explanation of what this method does..."
            ></textarea>
          </label>

          <label class="field">
            <span>Usage</span>
            <textarea
              bind:value={form.usage}
              name="usage"
              rows="5"
              placeholder="Explain when to use it or add a practical example..."
            ></textarea>
          </label>

          <div class="modal-actions">
            <button class="secondary-button" type="button" on:click={closeFormModal}>Cancel</button>
            <button class="primary-button" type="submit">
              {editingId ? 'Save Changes' : 'Save Method'}
            </button>
          </div>
        </form>
      </div>
    </div>
  {/if}

  {#if detailItemId}
    <div class="modal-backdrop" role="presentation" on:click={closeDetailModal}>
      <div
        class="modal-card"
        role="dialog"
        aria-modal="true"
        aria-labelledby="detail-modal-title"
        on:click|stopPropagation
        on:keydown|stopPropagation
        tabindex="-1"
      >
        {#if detailItem}
          <div class="modal-header">
            <div>
              <p class="eyebrow">Method Details</p>
              <h2 id="detail-modal-title">{detailItem.methodName}</h2>
            </div>
            <button class="icon-button" type="button" on:click={closeDetailModal} aria-label="Close detail dialog">
              ×
            </button>
          </div>

          <div class="detail-layout">
            <div class="detail-card">
              <p class="card-label">Programming Language</p>
              <strong>{detailItem.programmingLanguage}</strong>
            </div>

            <button class="detail-card detail-card--button" type="button" on:click={() => openTableModal(detailItem)}>
              <p class="card-label">Data Type</p>
              <strong>{detailItem.dataType}</strong>
              <span>Open structured table view</span>
            </button>
          </div>

          <div class="detail-block">
            <p class="card-label">Description</p>
            <p>{detailItem.description}</p>
          </div>

          <div class="detail-block">
            <p class="card-label">Usage</p>
            <p>{detailItem.usage}</p>
          </div>

          <div class="modal-actions">
            <button class="secondary-button" type="button" on:click={() => openTableModal(detailItem)}>
              Open Table View
            </button>
            <button class="ghost-button" type="button" on:click={() => openEditFromDetail(detailItem)}>
              Edit
            </button>
            <button class="danger-button" type="button" on:click={() => askDeleteFromDetail(detailItem.id)}>
              Delete
            </button>
          </div>
        {/if}
      </div>
    </div>
  {/if}

  {#if tableItemId}
    <div class="modal-backdrop" role="presentation" on:click={closeTableModal}>
      <div
        class="modal-card modal-card--table"
        role="dialog"
        aria-modal="true"
        aria-labelledby="table-modal-title"
        on:click|stopPropagation
        on:keydown|stopPropagation
        tabindex="-1"
      >
        {#if tableItem}
          <div class="modal-header">
            <div>
              <p class="eyebrow">Structured Table View</p>
              <h2 id="table-modal-title">{tableItem.dataType}</h2>
            </div>
            <button class="icon-button" type="button" on:click={closeTableModal} aria-label="Close table dialog">
              ×
            </button>
          </div>

          <div class="table-shell">
            <table class="detail-table">
              <thead>
                <tr>
                  <th>Method Name</th>
                  <th>Programming Language</th>
                  <th>Data Type</th>
                  <th>Description</th>
                  <th>Usage</th>
                </tr>
              </thead>
              <tbody>
                <tr>
                  <td>{tableItem.methodName}</td>
                  <td>{tableItem.programmingLanguage}</td>
                  <td>{tableItem.dataType}</td>
                  <td>{tableItem.description}</td>
                  <td>{tableItem.usage}</td>
                </tr>
              </tbody>
            </table>
          </div>
        {/if}
      </div>
    </div>
  {/if}

  {#if deleteTargetId}
    <div class="modal-backdrop" role="presentation" on:click={closeDeleteModal}>
      <div
        class="modal-card modal-card--compact"
        role="dialog"
        aria-modal="true"
        aria-labelledby="delete-modal-title"
        on:click|stopPropagation
        on:keydown|stopPropagation
        tabindex="-1"
      >
        <div class="modal-header">
          <div>
            <p class="eyebrow">Delete Method</p>
            <h2 id="delete-modal-title">Remove this method?</h2>
          </div>
          <button class="icon-button" type="button" on:click={closeDeleteModal} aria-label="Close delete dialog">
            ×
          </button>
        </div>

        <p class="dialog-copy">
          "{getDeleteTarget()?.methodName}" will be permanently removed from your saved methods.
        </p>

        <div class="modal-actions">
          <button class="secondary-button" type="button" on:click={closeDeleteModal}>Cancel</button>
          <button class="danger-button" type="button" on:click={confirmDelete}>Delete Method</button>
        </div>
      </div>
    </div>
  {/if}
</main>
