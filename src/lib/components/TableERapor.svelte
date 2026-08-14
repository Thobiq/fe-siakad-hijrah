<script lang="ts">
  export let data = [];
  export let isLoading = false;
  export let tingkat = ""; // ex: "kb", "tk-a", "tk-b"
  
  // Fitur Pencarian
  let searchQuery = "";
  
  // State Loading untuk Download PDF
  let downloadingId = null;

  $: displayedData = data.filter(item => {
    if (!searchQuery) return true;
    const term = searchQuery.toLowerCase();
    return (
      item.nama?.toLowerCase().includes(term) ||
      item.noInduk?.toLowerCase().includes(term) ||
      item.kelas?.toLowerCase().includes(term)
    );
  });

  import { PUBLIC_API_URL } from '$env/static/public';

  async function deleteRapor(raporId, rowId) {
    if (!confirm("Apakah Anda yakin ingin menghapus data rapor ini? Semua narasi dan foto akan hilang.")) return;
    
    const token = localStorage.getItem('auth_token');
    try {
      const response = await fetch(`${PUBLIC_API_URL}/rapor/${raporId}`, {
        method: 'DELETE',
        headers: {
          'Authorization': `Bearer ${token}`,
          'Accept': 'application/json'
        }
      });
      const result = await response.json();
      if (response.ok && result.status) {
        alert("Data rapor berhasil dihapus!");
        // Update state lokal agar tidak perlu refresh halaman
        data = data.map(item => item.id === rowId ? { ...item, rapor_id: null } : item);
      } else {
        alert("Gagal menghapus rapor: " + result.message);
      }
    } catch (err) {
      console.error(err);
      alert("Terjadi kesalahan jaringan.");
    }
  }

  async function downloadPdf(raporId, namaSiswa) {
    if (downloadingId === raporId) return; // Mencegah double click
    downloadingId = raporId;
    const token = localStorage.getItem('auth_token');
    try {
      const response = await fetch(`${PUBLIC_API_URL}/rapor/${raporId}/pdf`, {
        method: 'GET',
        headers: {
          'Authorization': `Bearer ${token}`
        }
      });

      if (!response.ok) {
        alert("Gagal mengunduh rapor.");
        downloadingId = null;
        return;
      }

      const blob = await response.blob();
      const url = window.URL.createObjectURL(blob);
      const a = document.createElement('a');
      a.href = url;
      // Nama file akan disesuaikan dengan header Content-Disposition jika ada,
      // tetapi kita juga bisa menset default di sini
      a.download = `Rapor_TK_${namaSiswa.replace(/ /g, '_')}.pdf`;
      document.body.appendChild(a);
      a.click();
      document.body.removeChild(a);
      window.URL.revokeObjectURL(url);
    } catch (err) {
      console.error(err);
      alert("Terjadi kesalahan jaringan.");
    } finally {
      downloadingId = null;
    }
  }
</script>

<div class="bg-white rounded-3xl shadow-[0_4px_20px_-10px_rgba(0,0,0,0.05)] border border-gray-100 p-6 md:p-8 flex flex-col gap-6">
  
  <div class="flex flex-col md:flex-row md:items-center justify-between gap-4">
    <h2 class="text-xl font-bold text-gray-700">Daftar Siswa (Penilaian Selesai)</h2>
    <div class="relative w-full md:w-[300px]">
      <input type="text" bind:value={searchQuery} placeholder="Cari siswa..." class="w-full pl-5 pr-10 py-2.5 rounded-xl border border-gray-300 focus:ring-2 focus:ring-[#2da76b] focus:border-[#2da76b] outline-none text-gray-700 transition-all"/>
      <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="absolute right-4 top-1/2 -translate-y-1/2 text-gray-400"><circle cx="11" cy="11" r="8"/><path d="m21 21-4.3-4.3"/></svg>
    </div>
  </div>

  <div class="rounded-2xl overflow-x-auto border border-gray-100">
    <table class="w-full text-left border-collapse whitespace-nowrap">
      <thead>
        <tr class="bg-[#2da76b] text-white">
          <th class="px-6 py-4 font-semibold text-sm">Nama Siswa</th>
          <th class="px-6 py-4 font-semibold text-sm">No. Induk</th>
          <th class="px-6 py-4 font-semibold text-sm">Kelas</th>
          <th class="px-6 py-4 font-semibold text-sm text-center">Aksi</th>
        </tr>
      </thead>
      <tbody>
        {#if isLoading}
          <tr>
            <td colspan="4" class="px-6 py-8 text-center text-gray-500 font-medium">
              <div class="flex items-center justify-center gap-3">
                <svg class="animate-spin h-6 w-6 text-[#2da76b]" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24"><circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle><path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path></svg>
                Memuat data...
              </div>
            </td>
          </tr>
        {:else}
          {#each displayedData as row}
            <tr class="odd:bg-white even:bg-gray-50 hover:bg-green-50/50 transition-colors">
              <td class="px-6 py-4 font-bold text-gray-700">{row.nama}</td>
              <td class="px-6 py-4 text-gray-600">{row.noInduk}</td>
              <td class="px-6 py-4 text-gray-600">{row.kelas}</td>
              <td class="px-6 py-4 flex items-center justify-center gap-2">
                {#if row.rapor_id}
                  <a href="/e-rapor/{tingkat.toLowerCase().replace(' ', '-')}/buat/{row.id}" class="bg-blue-500 hover:bg-blue-600 text-white px-4 py-2 rounded-xl font-bold transition-colors shadow-sm text-sm flex items-center gap-2">
                    <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M17 3a2.85 2.83 0 1 1 4 4L7.5 20.5 2 22l1.5-5.5Z"/><path d="m15 5 4 4"/></svg>
                    Edit
                  </a>
                  <button on:click={() => downloadPdf(row.rapor_id, row.nama)} disabled={downloadingId === row.rapor_id} class="bg-gray-700 hover:bg-gray-800 disabled:bg-gray-400 text-white px-4 py-2 rounded-xl font-bold transition-colors shadow-sm text-sm flex items-center gap-2">
                    {#if downloadingId === row.rapor_id}
                      <svg class="animate-spin h-4 w-4 text-white" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24"><circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle><path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path></svg>
                      Loading...
                    {:else}
                      <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"/><polyline points="7 10 12 15 17 10"/><line x1="12" x2="12" y1="15" y2="3"/></svg>
                      Download PDF
                    {/if}
                  </button>
                  <button on:click={() => deleteRapor(row.rapor_id, row.id)} class="bg-red-500 hover:bg-red-600 text-white px-4 py-2 rounded-xl font-bold transition-colors shadow-sm text-sm flex items-center gap-2">
                    <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M3 6h18"/><path d="M19 6v14c0 1-1 2-2 2H7c-1 0-2-1-2-2V6"/><path d="M8 6V4c0-1 1-2 2-2h4c1 0 2 1 2 2v2"/><line x1="10" x2="10" y1="11" y2="17"/><line x1="14" x2="14" y1="11" y2="17"/></svg>
                    Hapus
                  </button>
                {:else}
                  <a href="/e-rapor/{tingkat.toLowerCase().replace(' ', '-')}/buat/{row.id}" class="bg-[#2da76b] hover:bg-[#289562] text-white px-4 py-2 rounded-xl font-bold transition-colors shadow-sm text-sm">
                    Buat Rapor
                  </a>
                {/if}
              </td>
            </tr>
          {/each}
          {#if displayedData.length === 0}
            <tr>
              <td colspan="4" class="px-6 py-8 text-center text-gray-400 font-medium">Tidak ada data siswa yang penilaiannya sudah selesai.</td>
            </tr>
          {/if}
        {/if}
      </tbody>
    </table>
  </div>
</div>
