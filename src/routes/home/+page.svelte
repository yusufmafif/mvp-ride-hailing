<script>
  import { onMount } from 'svelte';
  import { goto } from '$app/navigation';
  import { supabase } from '$lib/supabaseClient';
  import maplibregl from 'maplibre-gl';
  // Corrected import path for MapLibre GL CSS
  import 'maplibre-gl/dist/maplibre-gl.css';

  let user = null;
  let pickup = '';
  let destination = '';
  let suggestionsPickup = [];
  let suggestionsDestination = [];
  let pickupCoord = null;
  let destinationCoord = null;
let routeInfo = null;
let showRoutePanel = false;
  let map;
  let currentLocation = null;
  let isSearchingDriver = false; // New state for driver search loading
  let driverFoundMessage = ''; // New state for driver search message

    /**
     * @type {maplibregl.Marker | null}
     */
let pickupMarker = null; // Marker untuk lokasi pickup

    /**
     * @type {maplibregl.Marker | null}
     */
let destinationMarker = null; // Marker untuk lokasi destinasi

  const TARIF_PER_KM = 2000; // Tariff per kilometer

  // Function to use current location as pickup
  const gunakanLokasiSekarang = async () => {
    if (!currentLocation) {
      driverFoundMessage = 'Lokasi saat ini tidak tersedia.';
      return;
    }
    const [lon, lat] = currentLocation;

    try {
      // Reverse geocoding to get address from coordinates
      const res = await fetch(`https://nominatim.openstreetmap.org/reverse?format=json&lat=${lat}&lon=${lon}`);
      const data = await res.json();
      pickup = data.display_name;
      pickupCoord = [lon, lat];
      driverFoundMessage = ''; // Clear any previous messages

      // Hapus marker sebelumnya jika ada
      if (pickupMarker) {
        pickupMarker.remove();
      }
      // Tambahkan marker baru di lokasi pickup
      if (map) {
        pickupMarker = new maplibregl.Marker({ color: '#007bff' })
          .setLngLat([lon, lat])
          .addTo(map);
        map.flyTo({ center: [lon, lat], zoom: map.getZoom() });
      }
    } catch (error) {
      console.error("Error fetching current location address:", error);
      driverFoundMessage = 'Gagal mendapatkan alamat dari lokasi saat ini.';
    }
  };

  // Lifecycle hook: runs when the component is mounted
  onMount(async () => {
    // Check user session with Supabase
    const { data: { session } = {} } = await supabase.auth.getSession(); // Destructure with default empty object
    if (!session) {
      // If no session, redirect to login page
      return goto('/login');
    }
    user = session.user;

    // Get current geolocation
    navigator.geolocation.getCurrentPosition(async pos => {
      currentLocation = [pos.coords.longitude, pos.coords.latitude];
      // Initialize MapLibre GL map
      map = new maplibregl.Map({
        container: 'map', // HTML element ID for the map
        style: 'https://api.maptiler.com/maps/streets-v2/style.json?key=sfvOnNJOOuOfsE6IJMSR', // Map style
        center: currentLocation, // Center map at current location
        zoom: 13 // Initial zoom level
      });
      // Add navigation controls to the map
      map.addControl(new maplibregl.NavigationControl(), 'top-right');

      // Set pickup default ke lokasi sekarang (reverse geocode)
      try {
        const res = await fetch(`https://nominatim.openstreetmap.org/reverse?format=json&lat=${pos.coords.latitude}&lon=${pos.coords.longitude}`);
        const data = await res.json();
        pickup = data.display_name;
        pickupCoord = [pos.coords.longitude, pos.coords.latitude];
        // Tambahkan marker pickup
        if (pickupMarker) pickupMarker.remove();
        pickupMarker = new maplibregl.Marker({ color: '#007bff' })
          .setLngLat([pos.coords.longitude, pos.coords.latitude])
          .addTo(map);
      } catch (e) {
        // fallback: kosongkan pickup
        pickup = '';
        pickupCoord = null;
      }
      // Hapus marker destinasi sebelumnya jika ada
      if (destinationMarker) {
        destinationMarker.remove();
      }
      // Tambahkan marker baru di lokasi destinasi
      if (map && destinationCoord) {
        destinationMarker = new maplibregl.Marker({ color: '#28a745' })
          .setLngLat(destinationCoord)
          .addTo(map);
      }
    }, err => {
      // Alert user if geolocation is not enabled
      driverFoundMessage = "Aktifkan lokasi/GPS untuk menggunakan aplikasi.";
      console.error("Geolocation error:", err);
    });
  });

  // Function to fetch address suggestions from OpenStreetMap Nominatim
  const fetchSuggestions = async (query, type) => {
    if (!currentLocation || !query) {
      if (type === 'pickup') suggestionsPickup = [];
      else suggestionsDestination = [];
      return;
    }
    const [lon, lat] = currentLocation;
    // The delta value below roughly corresponds to a 60km bounding box around the current location.
    // This helps in getting more relevant local suggestions.
    const delta = 0.54;
    const viewbox = `${lon - delta},${lat - delta},${lon + delta},${lat + delta}`; // Bounding box for search
    try {
      const res = await fetch(`https://nominatim.openstreetmap.org/search?format=json&q=${query}&bounded=1&viewbox=${viewbox}&limit=5`);
      const data = await res.json();
      if (type === 'pickup') suggestionsPickup = data;
      else suggestionsDestination = data;
    } catch (error) {
      console.error("Error fetching suggestions:", error);
      if (type === 'pickup') suggestionsPickup = [];
      else suggestionsDestination = [];
    }
  };

  // Function to select a suggestion and update input fields and coordinates
  const selectSuggestion = (item, type) => {
    if (type === 'pickup') {
      pickup = item.display_name;
      pickupCoord = [parseFloat(item.lon), parseFloat(item.lat)];
      suggestionsPickup = []; // Clear suggestions after selection
      // Update pickup marker
      if (pickupMarker) pickupMarker.remove();
      if (map && pickupCoord) {
        pickupMarker = new maplibregl.Marker({ color: '#007bff' })
          .setLngLat(pickupCoord)
          .addTo(map);
        map.flyTo({ center: pickupCoord, zoom: map.getZoom() });
      }
    } else {
      destination = item.display_name;
      destinationCoord = [parseFloat(item.lon), parseFloat(item.lat)];
      suggestionsDestination = []; // Clear suggestions after selection
    }
  };

  // Function to find route using OSRM
const cariRute = async () => {
  if (!pickupCoord || !destinationCoord) {
    driverFoundMessage = 'Mohon lengkapi lokasi penjemputan dan tujuan.';
    return;
  }
  driverFoundMessage = '';
  isSearchingDriver = false;

  const url = `https://router.project-osrm.org/route/v1/driving/${pickupCoord.join(',')};${destinationCoord.join(',')}?overview=full&geometries=geojson`;
  try {
    const res = await fetch(url);
    const data = await res.json();

    if (data.routes && data.routes.length > 0) {
      const route = data.routes[0];
      const coords = route.geometry.coordinates;
      const bounds = new maplibregl.LngLatBounds();
      for (const coord of coords) bounds.extend(coord);
      if (map.getSource('route')) {
        map.removeLayer('route');
        map.removeSource('route');
      }
      map.addSource('route', {
        type: 'geojson',
        data: {
          type: 'Feature',
          geometry: { type: 'LineString', coordinates: coords }
        }
      });
      map.addLayer({
        id: 'route',
        type: 'line',
        source: 'route',
        layout: { 'line-join': 'round', 'line-cap': 'round' },
        paint: { 'line-color': '#ff0000', 'line-width': 4 }
      });
      map.fitBounds(bounds, { padding: 40 });
      if (destinationMarker) destinationMarker.remove();
      if (map && destinationCoord) {
        destinationMarker = new maplibregl.Marker({ color: '#28a745' })
          .setLngLat(destinationCoord)
          .addTo(map);
      }
      const jarakKm = route.distance / 1000;
      const waktu = Math.round(route.duration / 60);
      routeInfo = {
        distance: `${jarakKm.toFixed(2)} km`,
        duration: `${waktu} menit`,
        tarif: `Rp ${(jarakKm * TARIF_PER_KM).toLocaleString('id-ID', { minimumFractionDigits: 0, maximumFractionDigits: 1 })}`
      };
      showRoutePanel = true;
    } else {
      routeInfo = null;
      driverFoundMessage = 'Tidak dapat menemukan rute. Coba lokasi lain.';
    }
  } catch (error) {
    console.error("Error finding route:", error);
    routeInfo = null;
    driverFoundMessage = 'Terjadi kesalahan saat mencari rute.';
  }
};

function backToForm() {
  showRoutePanel = false;
}

  // New function to simulate finding a driver
const cariDriver = async () => {
    if (!routeInfo) {
      driverFoundMessage = 'Mohon cari rute terlebih dahulu.';
      return;
    }

    isSearchingDriver = true;
    driverFoundMessage = 'Mencari driver terdekat...';

    // Simulate API call delay for finding driver
    await new Promise(resolve => setTimeout(resolve, 2000)); // Shortened to 2 seconds for faster transition

    isSearchingDriver = false;

    // Simulate driver found and redirect to new page
    if (Math.random() > 0.1) { // Higher chance to find a driver for demonstration
        driverFoundMessage = 'Driver ditemukan! Mengarahkan ke halaman konfirmasi...';

        // Encode data to pass via URL search params (or a more robust state management if app grows)
        const dataToPass = {
            pickup,
            destination,
            pickupCoord,
            destinationCoord,
            routeInfo
        };
        const encodedData = encodeURIComponent(JSON.stringify(dataToPass));

        // Redirect to the new order confirmation page
        goto(`/order-confirm?data=${encodedData}`);

    } else {
        driverFoundMessage = 'Maaf, tidak ada driver yang tersedia saat ini. Coba lagi nanti.';
    }
  };

  // Function to handle user logout
  const handleLogout = async () => {
    await supabase.auth.signOut();
    goto('/login'); // Redirect to login page after logout
  };
</script>

<style>
  /* Basic styling for the container */
  .home-container {
    font-family: 'Inter', 'Roboto', sans-serif;
    width: 100vw;
    height: 100vh;
    margin: 0;
    background: #f8f9fa;
    min-height: 100vh;
    display: flex;
    flex-direction: column;
    overflow: hidden;
  }
  /* Header styling */
  .header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 0.7rem 1rem 0.7rem 1rem;
    background: #f8f9fa;
    border-bottom: 1px solid #e2e8f0;
    z-index: 10;
  }
  .header strong {
    color: #222;
    font-size: 1rem;
    font-weight: 500;
    letter-spacing: 0.2px;
  }
  /* Map placeholder styling */
  .map-placeholder {
    flex: 1;
    width: 100vw;
    height: 100%;
    background: #e9ecef;
    display: flex;
    justify-content: center;
    align-items: center;
    margin: 0;
    overflow: hidden;
    position: relative;
    z-index: 1;
  }
  .map-placeholder p {
    color: #6c757d;
  }
  /* Form input styling */
  .order-form {
    width: 100vw;
    max-width: 400px;
    margin: 0 auto;
    background: #f8f9fa;
    padding: 0.7rem 0.5rem 0.5rem 0.5rem;
    position: relative;
    z-index: 10;
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
    overflow-y: visible;
    border: none;
    box-shadow: none;
    border-radius: 0;
  }
  .order-form input {
    width: 100%;
    padding: 10px 10px;
    margin-bottom: 0.5rem;
    font-size: 0.95rem;
    border: 1px solid #e2e8f0;
    border-radius: 4px;
    background: #fff;
    transition: border-color 0.2s;
    box-sizing: border-box;
  }
  .order-form input:focus {
    border-color: #007bff;
    outline: none;
  }
  /* Button styling */
  .button-group {
    display: flex;
    gap: 10px; /* Space between buttons */
    margin-bottom: 1rem;
  }
  .order-button, .cari-driver-button, .gunakan-lokasi-button {
    width: 100%;
    padding: 10px 0;
    font-size: 0.95rem;
    color: #fff;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    background: #007bff;
    font-weight: 400;
    margin-bottom: 0.5rem;
    box-shadow: none;
    transition: background 0.2s;
  }
  .order-button {
    background: #007bff;
  }
  .order-button:hover {
    background: #0056b3;
  }
  .cari-driver-button {
    background: #28a745;
  }
  .cari-driver-button:hover {
    background: #218838;
  }
  .cari-driver-button:disabled {
    background: #adb5bd;
    cursor: not-allowed;
  }
  .gunakan-lokasi-button {
    background: #6c757d;
    font-size: 0.95rem;
    margin-bottom: 0.5rem;
    border:color(from color srgb r g b);
  }
  .gunakan-lokasi-button:hover {
    background: #5a6268;
  }
  .logout-button {
    background: transparent;
    color: #dc3545;
    border: none;
    padding: 0.3rem 0.7rem;
    border-radius: 4px;
    cursor: pointer;
    font-weight: 500;
    font-size: 0.95rem;
    transition: background 0.2s;
  }
  .logout-button:hover {
    background: #f8d7da;
  }
  /* Suggestions dropdown styling */
  .suggestions {
    background: #fff;
    border: 1px solid #e2e8f0;
    border-radius: 4px;
    position: absolute;
    left: 0;
    right: 0;
    top: 100%;
    z-index: 999;
    max-height: 180px;
    overflow-y: auto;
    width: 100%;
    box-shadow: none;
    margin-top: 2px;
  }
  .suggestions div {
    padding: 10px 10px;
    cursor: pointer;
    border-bottom: 1px solid #f2f2f2;
    font-size: 0.95rem;
    color: #222;
    transition: background 0.15s;
  }
  .suggestions div:last-child {
    border-bottom: none;
  }
  .suggestions div:hover {
    background: #f1f3f4;
  }
  /* Route info styling */
  .route-info {
    background: #f8f9fa;
    padding: 0.7rem 0.5rem;
    border-radius: 4px;
    box-shadow: none;
    border: none;
    font-size: 0.95rem;
    font-weight: 400;
    color: #222;
  }
  .route-info p {
    margin-bottom: 0.3rem;
    color: #222;
    font-size: 0.95rem;
  }
  .route-info p:last-child {
    margin-bottom: 0;
  }
  /* Message display styling */
  .message {
    margin-top: 1rem;
    padding: 0.7rem 0.5rem;
    border-radius: 4px;
    background: #f1f3f4;
    color: #444;
    border: none;
    text-align: center;
    font-weight: 400;
    font-size: 0.95rem;
    box-shadow: none;
  }
  .message.error {
    background: #fbeaea;
    color: #c82333;
  }
  .message.success {
    background: #eafbe9;
    color: #218838;
  }
  /* Responsive adjustments */
  @media (max-width: 600px) {
    .home-container {
      padding: 0;
    }
    .order-form {
      padding: 1rem 0.5rem 0.5rem 0.5rem;
      max-width: 98vw;
      margin: 0 auto;
      border-radius: 0 0 12px 12px;
    }
    .button-group {
      flex-direction: column;
    }
    .order-button, .cari-driver-button, .gunakan-lokasi-button {
      width: 100%;
      min-width: 0;
      max-width: 100%;
      flex: none;
      font-size: 0.98rem;
      padding: 11px 0;
    }
    .suggestions {
      width: 100%;
    }
    .route-info {
      padding: 1rem 0.5rem;
      font-size: 1rem;
    }
  }
</style>

{#if user}
<div class="home-container">
  <header class="header">
    <div>
      <strong>Halo, {user.email.split('@')[0]}!</strong>
    </div>
    <button class="logout-button" on:click={handleLogout}>Logout</button>
  </header>

  <div class="map-placeholder" id="map">
    <p>Memuat peta...</p>
  </div>

  {#if !showRoutePanel}
    <div class="order-form" style="position: relative;">
      <div style="position:relative; display:flex; align-items:center;">
        <input
          type="text"
          placeholder="Lokasi Penjemputan"
          bind:value={pickup}
          on:input={(e) => fetchSuggestions(e.target.value, 'pickup')}
          class="rounded-lg"
          style="flex:1; padding-right:2.2rem;"
        />
        <button
          type="button"
          on:click={gunakanLokasiSekarang}
          class="gunakan-lokasi-button"
          aria-label="Gunakan lokasi sekarang"
          style="position:absolute; right:0.3rem; background:transparent; border:none; color:#007bff; font-size:1.3rem; padding:0; height:1.8rem; width:1.8rem; display:flex; align-items:center; justify-content:center; cursor:pointer;"
        >
          <span aria-hidden="true">📍</span>
        </button>
        {#if suggestionsPickup.length}
          <div class="suggestions rounded-lg">
            {#each suggestionsPickup as item}
              <div on:click={() => selectSuggestion(item, 'pickup')}>{item.display_name}</div>
            {/each}
          </div>
        {/if}
      </div>
      <div style="position:relative;">
        <input
          type="text"
          placeholder="Lokasi Tujuan"
          bind:value={destination}
          on:input={(e) => fetchSuggestions(e.target.value, 'destination')}
          class="rounded-lg"
        />
        {#if suggestionsDestination.length}
          <div class="suggestions rounded-lg">
            {#each suggestionsDestination as item}
              <div on:click={() => selectSuggestion(item, 'destination')}>{item.display_name}</div>
            {/each}
          </div>
        {/if}
      </div>
      <button class="order-button rounded-lg" on:click={cariRute}>
        Cari Rute & Estimasi
      </button>
    </div>
  {/if}

  {#if showRoutePanel && routeInfo}
    <div class="route-info route-panel rounded-lg" style="position:relative; max-width:400px; margin:rem auto 0 auto;">
      <button on:click={backToForm} aria-label="Kembali ke form" style="position:absolute; left:0.5rem; top:0.5rem; background:none; border:none; font-size:1.1rem; color:#007bff; cursor:pointer; padding:0.18rem 0.6rem 0.18rem 0.18rem; z-index:2; line-height:1;">
        ←
      </button>
      <div style="text-align:center; padding-top:0.2rem; padding-left:2.2rem;">
        <p style="margin-top:0.2rem;"><strong>Jarak:</strong> {routeInfo.distance}</p>
        <p><strong>Waktu Tempuh:</strong> {routeInfo.duration}</p>
        <p><strong>Tarif:</strong> {routeInfo.tarif}</p>
      </div>
      <button
        class="cari-driver-button rounded-lg"
        on:click={cariDriver}
        disabled={isSearchingDriver}
        style="margin-top: 1.2rem; width: 100%;"
      >
        {#if isSearchingDriver}
          Mencari driver...
        {:else}
          Cari Driver
        {/if}
      </button>
    </div>
  {/if}

  {#if driverFoundMessage}
    <div class="message {driverFoundMessage.includes('Driver ditemukan') ? 'success' : driverFoundMessage.includes('Aktifkan lokasi') || driverFoundMessage.includes('Mohon lengkapi') ? '' : 'error'} rounded-lg">
      {driverFoundMessage}
    </div>
  {/if}
</div>
{/if}