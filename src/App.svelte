<script>
  const storageKey = 'method-catatan-items'

  const createEmptyForm = () => ({
    namaMethod: '',
    deskripsi: '',
    caraPenggunaan: '',
  })

  const initialItems = [
    {
      id: crypto.randomUUID(),
      namaMethod: 'Array.map()',
      deskripsi: 'Membuat array baru dari hasil transformasi setiap item pada array asal.',
      caraPenggunaan:
        'Gunakan ketika ingin mengubah isi array tanpa memodifikasi array aslinya, misalnya mengubah daftar angka menjadi string.',
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
      return Array.isArray(parsedItems) ? parsedItems : initialItems
    } catch (error) {
      window.localStorage.setItem(storageKey, JSON.stringify(initialItems))
      return initialItems
    }
  }

  let items = loadItems()
  let form = createEmptyForm()
  let editingId = null

  const syncItems = (nextItems) => {
    items = nextItems

    if (typeof window !== 'undefined') {
      window.localStorage.setItem(storageKey, JSON.stringify(nextItems))
    }
  }

  const resetForm = () => {
    form = createEmptyForm()
    editingId = null
  }

  const submitForm = () => {
    const payload = {
      namaMethod: form.namaMethod.trim(),
      deskripsi: form.deskripsi.trim(),
      caraPenggunaan: form.caraPenggunaan.trim(),
    }

    if (!payload.namaMethod || !payload.deskripsi || !payload.caraPenggunaan) {
      window.alert('Semua field wajib diisi.')
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
    } else {
      syncItems([
        {
          id: crypto.randomUUID(),
          ...payload,
          createdAt: new Date().toISOString(),
        },
        ...items,
      ])
    }

    resetForm()
  }

  const startEdit = (item) => {
    editingId = item.id
    form = {
      namaMethod: item.namaMethod,
      deskripsi: item.deskripsi,
      caraPenggunaan: item.caraPenggunaan,
    }
  }

  const deleteItem = (id) => {
    const targetItem = items.find((item) => item.id === id)

    if (!targetItem) {
      return
    }

    const confirmed = window.confirm(`Hapus "${targetItem.namaMethod}"?`)

    if (!confirmed) {
      return
    }

    syncItems(items.filter((item) => item.id !== id))

    if (editingId === id) {
      resetForm()
    }
  }

  const exportToPdf = () => {
    if (!items.length) {
      window.alert('Belum ada data yang bisa diexport.')
      return
    }

    const printWindow = window.open('', '_blank', 'width=960,height=720')

    if (!printWindow) {
      window.alert('Popup diblokir browser. Izinkan popup lalu coba lagi.')
      return
    }

    const exportedAt = new Date().toLocaleString('id-ID', {
      dateStyle: 'full',
      timeStyle: 'short',
    })

    const cards = items
      .map(
        (item, index) => `
          <article class="print-card">
            <div class="print-number">Method ${index + 1}</div>
            <h2>${escapeHtml(item.namaMethod)}</h2>
            <section>
              <h3>Deskripsi</h3>
              <p>${escapeHtml(item.deskripsi).replace(/\n/g, '<br>')}</p>
            </section>
            <section>
              <h3>Cara Penggunaan</h3>
              <p>${escapeHtml(item.caraPenggunaan).replace(/\n/g, '<br>')}</p>
            </section>
          </article>
        `,
      )
      .join('')

    printWindow.document.write(`
      <!doctype html>
      <html lang="id">
        <head>
          <meta charset="UTF-8" />
          <title>Export Method Catatan</title>
          <style>
            :root {
              color-scheme: light;
              font-family: Arial, Helvetica, sans-serif;
            }

            * {
              box-sizing: border-box;
            }

            body {
              margin: 0;
              padding: 40px;
              color: #111827;
              background: #f8fafc;
            }

            .print-shell {
              max-width: 960px;
              margin: 0 auto;
            }

            .print-header {
              margin-bottom: 28px;
              padding: 24px 28px;
              border-radius: 20px;
              background: linear-gradient(135deg, #111827, #1f2937 60%, #374151);
              color: #f9fafb;
            }

            .print-header h1 {
              margin: 0 0 8px;
              font-size: 28px;
            }

            .print-header p {
              margin: 0;
              color: #d1d5db;
              line-height: 1.6;
            }

            .print-list {
              display: grid;
              gap: 18px;
            }

            .print-card {
              padding: 24px 28px;
              border-radius: 18px;
              background: #ffffff;
              border: 1px solid #dbe4f0;
              break-inside: avoid;
              page-break-inside: avoid;
            }

            .print-number,
            .print-card h3 {
              font-size: 12px;
              letter-spacing: 0.12em;
              text-transform: uppercase;
              color: #475569;
            }

            .print-card h2 {
              margin: 10px 0 18px;
              font-size: 24px;
              color: #0f172a;
            }

            .print-card section + section {
              margin-top: 18px;
            }

            .print-card h3 {
              margin: 0 0 8px;
            }

            .print-card p {
              margin: 0;
              line-height: 1.7;
              color: #334155;
            }

            @media print {
              body {
                padding: 0;
                background: #ffffff;
              }
            }
          </style>
        </head>
        <body>
          <main class="print-shell">
            <header class="print-header">
              <h1>Method Catatan</h1>
              <p>Dicetak pada ${escapeHtml(exportedAt)}.</p>
              <p>Total method: ${items.length}</p>
            </header>
            <section class="print-list">${cards}</section>
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
  <title>Method Catatan</title>
  <meta
    name="description"
    content="CRUD basic untuk mencatat nama method, deskripsi, dan cara penggunaan."
  />
</svelte:head>

<main class="page-shell">
  <section class="hero-panel">
    <div class="hero-copy">
      <p class="eyebrow">CRUD Basic</p>
      <h1>Catat method penting dan export isinya ke PDF.</h1>
      <p class="lead">
        Data disimpan langsung di browser tanpa database. Cocok buat catatan cepat soal
        method, fungsi, atau snippet yang sering dipakai.
      </p>
    </div>
    <div class="hero-stats">
      <div class="stat-card">
        <span>Total Catatan</span>
        <strong>{items.length}</strong>
      </div>
      <div class="stat-card">
        <span>Status Form</span>
        <strong>{editingId ? 'Edit Mode' : 'Tambah Baru'}</strong>
      </div>
    </div>
  </section>

  <section class="content-grid">
    <form class="editor-panel" on:submit|preventDefault={submitForm}>
      <div class="panel-heading">
        <div>
          <p class="eyebrow">Form Input</p>
          <h2>{editingId ? 'Edit Method' : 'Tambah Method'}</h2>
        </div>
        {#if editingId}
          <button class="ghost-button" type="button" on:click={resetForm}>Batal Edit</button>
        {/if}
      </div>

      <label class="field">
        <span>Tipe Data</span>
        <input
          bind:value={form.namaMethod}
          type="text"
          name="namaMethod"
          placeholder="Contoh: Int atau String"
        />
      </label>

      <label class="field">
        <span>Nama Method</span>
        <input
          bind:value={form.namaMethod}
          type="text"
          name="namaMethod"
          placeholder="Contoh: String.includes()"
        />
      </label>

      <label class="field">
        <span>Deskripsi</span>
        <textarea
          bind:value={form.deskripsi}
          name="deskripsi"
          rows="4"
          placeholder="Jelaskan fungsi method ini secara singkat..."
        ></textarea>
      </label>

      <label class="field">
        <span>Cara Penggunaan</span>
        <textarea
          bind:value={form.caraPenggunaan}
          name="caraPenggunaan"
          rows="5"
          placeholder="Tuliskan kapan dipakai atau contoh cara menggunakannya..."
        ></textarea>
      </label>

      <div class="form-actions">
        <button class="primary-button" type="submit">
          {editingId ? 'Update Method' : 'Simpan Method'}
        </button>
        <button class="secondary-button" type="button" on:click={exportToPdf}>
          Export ke PDF
        </button>
      </div>
    </form>

    <section class="list-panel">
      <div class="panel-heading">
        <div>
          <p class="eyebrow">Daftar Method</p>
          <h2>Semua Catatan</h2>
        </div>
      </div>

      {#if items.length}
        <div class="method-list">
          {#each items as item}
            <article class="method-card">
              <div class="card-head">
                <div>
                  <p class="card-label">Nama Method</p>
                  <h3>{item.namaMethod}</h3>
                </div>
                <div class="card-actions">
                  <button class="ghost-button" type="button" on:click={() => startEdit(item)}>
                    Edit
                  </button>
                  <button class="danger-button" type="button" on:click={() => deleteItem(item.id)}>
                    Hapus
                  </button>
                </div>
              </div>

              <div class="card-body">
                <div>
                  <p class="card-label">Deskripsi</p>
                  <p>{item.deskripsi}</p>
                </div>
                <div>
                  <p class="card-label">Cara Penggunaan</p>
                  <p>{item.caraPenggunaan}</p>
                </div>
              </div>
            </article>
          {/each}
        </div>
      {:else}
        <div class="empty-state">
          <p>Belum ada catatan method. Isi form di sebelah kiri untuk mulai menambahkan data.</p>
        </div>
      {/if}
    </section>
  </section>
</main>
