<script>
  import { onMount } from 'svelte';
  import { goto } from '$app/navigation';
  import { supabase } from '$lib/supabaseClient';

  // Simulasi delay splash screen (2.5 detik)
 onMount(() => {
    // Tampilkan splash screen selama 2 detik
    setTimeout(async () => {
      // Cek sesi pengguna di Supabase
      const { data: { session } } = await supabase.auth.getSession();

      if (session) {
        // Jika ada sesi (sudah login), arahkan ke halaman utama
        goto('/home');
      } else {
        // Jika tidak ada sesi, arahkan ke halaman login
        goto('/login');
      }
    }, 2000);
  });
</script>

<style>
    .splash-container {
    position: fixed;
    inset: 0; /* ini cara modern utk full kiri-kanan-atas-bawah */
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    background: linear-gradient(160deg, #03c04c 0%, #61a80f 100%);
    z-index: 9999;
  }

  .logo {
    width: 120px;
    height: 120px;
    margin-bottom: 20px;
  }

  .tagline {
    font-size: 25px;
    font-weight: 600;
    color: #ffffff;
    margin-bottom: 40px;
  }

  .loader {
    width: 40px;
    height: 40px;
    border: 4px solid #ffffff;
    border-top: 4px solid #0ACF83;
    border-radius: 50%;
    animation: spin 1s linear infinite;
  }

  @keyframes spin {
    0% { transform: rotate(0deg); }
    100% { transform: rotate(360deg); }
  }
</style>

<div class="splash-container">
  <img src="/logo-placeholder.png" alt="Logo" class="logo" />
  <div class="tagline">Kirim Barang Lebih Mudah</div>
  <div class="loader"></div>
</div>
