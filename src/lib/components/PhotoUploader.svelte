<script lang="ts">
  // Array of photo objects: { id: number, file: File | null, preview: string | null }
  export let photos = [
    { id: 1, file: null, preview: null },
    { id: 2, file: null, preview: null },
    { id: 3, file: null, preview: null }
  ];

  let fileInputs = [];

  function triggerFileInput(index) {
    if (fileInputs[index]) {
      fileInputs[index].click();
    }
  }

  function handleFileChange(event, index) {
    const file = event.target.files[0];
    if (file) {
      const reader = new FileReader();
      reader.onload = (e) => {
        photos[index] = {
          ...photos[index],
          file: file,
          preview: e.target.result
        };
      };
      reader.readAsDataURL(file);
    }
    // Reset input value so the same file can be selected again if removed
    event.target.value = null;
  }

  function removePhoto(index) {
    photos[index] = {
      ...photos[index],
      file: null,
      preview: null
    };
  }
</script>

<div class="flex gap-4 items-center">
  {#each photos as photo, i (photo.id)}
    <div class="relative w-32 h-32 rounded-2xl border-2 border-dashed {photo.preview ? 'border-transparent' : 'border-gray-300 hover:border-[#2da76b]'} flex items-center justify-center bg-gray-50 transition-all group overflow-hidden">
      
      {#if photo.preview}
        <img src={photo.preview} alt="Preview" class="w-full h-full object-cover rounded-xl" />
        
        <!-- Overlay untuk Hapus / Ubah -->
        <div class="absolute inset-0 bg-black/60 opacity-0 group-hover:opacity-100 transition-opacity flex flex-col items-center justify-center gap-2.5 rounded-xl p-3">
          <button type="button" on:click={() => triggerFileInput(i)} class="w-full text-white text-sm font-bold bg-white/20 px-4 py-2 rounded-xl hover:bg-white/30 backdrop-blur-sm transition-colors shadow-sm">
            Ubah
          </button>
          <button type="button" on:click={() => removePhoto(i)} class="w-full text-white text-sm font-bold bg-red-500/90 px-4 py-2 rounded-xl hover:bg-red-600 shadow-sm transition-colors">
            Hapus
          </button>
        </div>
      {:else}
        <button type="button" on:click={() => triggerFileInput(i)} class="w-full h-full flex flex-col items-center justify-center gap-2 text-gray-400 group-hover:text-[#2da76b] transition-colors">
          <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><rect width="18" height="18" x="3" y="3" rx="2" ry="2"/><circle cx="9" cy="9" r="2"/><path d="m21 15-3.086-3.086a2 2 0 0 0-2.828 0L6 21"/></svg>
          <span class="text-xs font-semibold">Tambah</span>
        </button>
      {/if}

      <!-- Hidden Input File -->
      <input 
        type="file" 
        accept="image/*" 
        bind:this={fileInputs[i]} 
        on:change={(e) => handleFileChange(e, i)} 
        class="hidden" 
      />
    </div>
  {/each}
</div>
