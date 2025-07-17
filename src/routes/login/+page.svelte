<script>
  import { goto } from '$app/navigation';
  import { supabase } from '$lib/supabaseClient';
  let email = '';
  let password = '';
  let loading = false;
  let errorMessage = '';



  const handleLogin = async () => {
    try {
      loading = true;
      errorMessage = '';
      const { error } = await supabase.auth.signInWithPassword({ email, password });
      if (error) throw error;
      goto('/home');
    } catch (error) {
      errorMessage = error.message;
    } finally {
      loading = false;
    }
  };

  const handleSignUp = async () => {
    try {
      loading = true;
      errorMessage = '';
      const { error } = await supabase.auth.signUp({ email, password });
      if (error) throw error;
      alert('Pendaftaran berhasil! Cek email Anda untuk verifikasi.');
    } catch (error) {
      errorMessage = error.message;
    } finally {
      loading = false;
    }
  };
</script>

<style>
  :global(html, body) {
    margin: 0;
    padding: 0;
    font-family: 'Segoe UI', Roboto, sans-serif;
    background: linear-gradient(to bottom, #0ACF83, #ffffff);
    height: 100%;
  }

  .container {
    min-height: 100vh;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: flex-start;
    padding: 40px 20px;
    background-color: white;
    border-top-left-radius: 50% 10%;
    border-top-right-radius: 50% 10%;
    margin-top: -100px;
  }

  .header {
    width: 100%;
    height: 230px;
    background: linear-gradient(180deg, #03c04c, #61a80f);
    display: flex;
    justify-content: center;
    align-items: center;
    border-bottom-left-radius: 50% 10%;
    border-bottom-right-radius: 50% 10%;
  }

  .logo {
    width: 80px;
    height: 80px;
  }

  h2 {
    margin: 30px 0 20px;
    color: #333;
  }

  .form-group {
    width: 100%;
    max-width: 300px;
    margin-bottom: 15px;
  }

  input {
    width: 100%;
    padding: 12px 15px;
    font-size: 1rem;
    border: 1px solid #ccc;
    border-radius: 25px;
    outline: none;
    background-color: #fff;
    box-sizing: border-box;
    transition: border-color 0.2s;
  }

  input:focus {
    border-color: #0ACF83;
    box-shadow: 0 0 0 2px rgba(10, 207, 131, 0.2);
  }

  .button {
    width: 100%;
    max-width: 300px;
    padding: 12px 0;
    font-size: 16px;
    color: #0ACF83;
    background: transparent;
    border: 2px solid #0ACF83;
    border-radius: 25px;
    cursor: pointer;
    margin-bottom: 12px;
    transition: all 0.2s ease-in-out;
  }

  .button:hover {
    background: #0ACF83;
    color: white;
  }

  .button:disabled {
    background: #ccc;
    color: #666;
    cursor: not-allowed;
  }

  .error {
    color: red;
    font-size: 14px;
    margin: 10px 0;
  }

  .footer-text {
    font-size: 14px;
    color: #444;
    margin-top: 10px;
  }

  .footer-text a {
    color: #0ACF83;
    text-decoration: none;
    margin-left: 5px;
  }
</style>

<div class="header">
  <img src="/logo-placeholder.png" alt="Logo" class="logo" />
</div>

<div class="container">
  <h2>Selamat Datang</h2>

  <div class="form-group">
    <input type="email" bind:value={email} placeholder="Email" />
  </div>

  <div class="form-group">
    <input type="password" bind:value={password} placeholder="Password" />
  </div>

  {#if errorMessage}
    <div class="error">{errorMessage}</div>
  {/if}

  <button class="button" on:click={handleLogin} disabled={loading}>
    {loading ? 'Memproses...' : 'Login'}
  </button>

  <button class="button" on:click={handleSignUp} disabled={loading}>
    {loading ? 'Memproses...' : 'Daftar'}
  </button>

  <div class="footer-text">
    Belum punya akun?
    <a href="#" on:click|preventDefault={handleSignUp}>Daftar di sini</a>
  </div>
</div>
