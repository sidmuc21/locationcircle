<script>
  import { onMount, onDestroy } from 'svelte';
  import {
    getDistance,
    getGreatCircleBearing,
    isPointWithinRadius
  } from 'geolib';

  let userCoords = { latitude: null, longitude: null };
  let alpha = 0;
  let beta = 0;
  let gamma = 0;

  const targetCoords = {
    latitude: 42.05913363079434,
    longitude: 19.51725368912721
  };

  let distanceToTarget = null;
  let bearingToTarget = 0;
  let heading = 0;
  let inRange = false;
  let geoWatchId;

  let needsOrientationPermission = false;
  let orientationPermissionGranted = false;

  // Battery state
  let batteryLevel = null;
  let batteryCharging = null;
  let battery = null;

  let hasVibrated = false;  // to prevent repeated vibrations

  function startGeolocation() {
    if (typeof navigator !== 'undefined' && navigator.geolocation) {
      geoWatchId = navigator.geolocation.watchPosition(
        (pos) => {
          userCoords.latitude = pos.coords.latitude;
          userCoords.longitude = pos.coords.longitude;

          if (userCoords.latitude && userCoords.longitude) {
            distanceToTarget = getDistance(userCoords, targetCoords);
            bearingToTarget = getGreatCircleBearing(userCoords, targetCoords);
            const currentlyInRange = isPointWithinRadius(userCoords, targetCoords, 5);

            // Vibrate on entering range once
            if (currentlyInRange && !inRange && navigator.vibrate) {
              navigator.vibrate(500);
              hasVibrated = true;
            }

            if (!currentlyInRange) {
              hasVibrated = false; // reset vibration flag if out of range
            }

            inRange = currentlyInRange;

            heading = (bearingToTarget - alpha + 360) % 360;
          }
        },
        (err) => {
          console.error('Geolocation error:', err);
        },
        { enableHighAccuracy: true, maximumAge: 1000, timeout: 5000 }
      );
    } else {
      alert('Geolocation not supported');
    }
  }

  function handleOrientation(event) {
    if (typeof event.alpha === 'number') alpha = event.alpha;
    if (typeof event.beta === 'number') beta = event.beta;
    if (typeof event.gamma === 'number') gamma = event.gamma;

    heading = (bearingToTarget - alpha + 360) % 360;
  }

  function requestOrientationPermission() {
    if (
      typeof DeviceOrientationEvent !== 'undefined' &&
      typeof DeviceOrientationEvent.requestPermission === 'function'
    ) {
      DeviceOrientationEvent.requestPermission()
        .then(state => {
          if (state === 'granted') {
            window.addEventListener('deviceorientation', handleOrientation, true);
            orientationPermissionGranted = true;
            needsOrientationPermission = false;
          } else {
            alert('Permission denied');
          }
        })
        .catch(console.error);
    }
  }

  function updateBatteryInfo() {
    if (!battery) return;
    batteryLevel = Math.round(battery.level * 100);
    batteryCharging = battery.charging;
  }

  onMount(() => {
    startGeolocation();

    if (
      typeof DeviceOrientationEvent !== 'undefined' &&
      typeof DeviceOrientationEvent.requestPermission === 'function'
    ) {
      needsOrientationPermission = true;
    } else if (typeof window !== 'undefined') {
      window.addEventListener('deviceorientation', handleOrientation, true);
      orientationPermissionGranted = true;
    }

    // Battery API
    if (navigator.getBattery) {
      navigator.getBattery().then(batt => {
        battery = batt;
        updateBatteryInfo();

        battery.addEventListener('levelchange', updateBatteryInfo);
        battery.addEventListener('chargingchange', updateBatteryInfo);
      });
    }
  });

  onDestroy(() => {
    if (typeof navigator !== 'undefined' && geoWatchId !== undefined) {
      navigator.geolocation.clearWatch(geoWatchId);
    }

    if (typeof window !== 'undefined') {
      window.removeEventListener('deviceorientation', handleOrientation);

      if (battery) {
        battery.removeEventListener('levelchange', updateBatteryInfo);
        battery.removeEventListener('chargingchange', updateBatteryInfo);
      }
    }
  });
</script>

<style>
  .container {
    width: 100vw;
    height: 100vh;
    background: #f9f9f9;
    display: flex;
    justify-content: center;
    align-items: center;
    position: relative;
    font-family: sans-serif;
  }

  .circle {
    width: 40vmin;
    height: 40vmin;
    border-radius: 50%;
    background-color: red;
    transition: background-color 0.3s ease;
    z-index: 1;
  }

  .circle.green {
    background-color: limegreen;
  }

  .arrow {
    position: absolute;
    top: 10vh;
    left: 50%;
    transform-origin: bottom center;
    width: 0;
    height: 0;
    margin-left: -3vmin;
    border-left: 3vmin solid transparent;
    border-right: 3vmin solid transparent;
    border-bottom: 9vmin solid black;
    z-index: 2;
    transition: transform 0.15s linear;
  }

  .info {
    position: absolute;
    bottom: 2vh;
    left: 2vw;
    background: rgba(255,255,255,0.9);
    padding: 1em;
    border-radius: 8px;
    font-size: 1rem;
    max-width: 300px;
    box-shadow: 0 0 10px rgba(0,0,0,0.1);
  }

  button.permission-button {
    position: absolute;
    top: 2vh;
    right: 2vw;
    padding: 0.6em 1em;
    background: #007bff;
    color: white;
    border: none;
    border-radius: 6px;
    cursor: pointer;
  }

  button.permission-button:hover {
    background: #0056b3;
  }
</style>

<div class="container">
  <div class="circle" class:green={inRange}></div>

  {#if !inRange}
    <div
      class="arrow"
      style="transform: rotate({heading}deg);"
      aria-label="Compass arrow pointing towards target"
    ></div>
  {/if}

  <div class="info" aria-live="polite">
    <p>📍 Lat: {userCoords.latitude ?? 'Loading...'}</p>
    <p>📍 Lon: {userCoords.longitude ?? 'Loading...'}</p>
    <p>📏 Abstand zum Ziel: {distanceToTarget !== null ? Math.round(distanceToTarget) : '...'} m</p>
    <p>🧭 Kurs zum Ziel: {bearingToTarget ? Math.round(bearingToTarget) : '...'}°</p>
    <p>📱 Alpha: {Math.round(alpha)}</p>
    <p>📱 Beta: {Math.round(beta)}</p>
    <p>📱 Gamma: {Math.round(gamma)}</p>
    <p>➡️ Angepasster Kurs: {Math.round(heading)}°</p>

    {#if batteryLevel !== null}
      <p>🔋 Akku: {batteryLevel}% {batteryCharging ? '(Lädt)' : '(Nicht lädt)'}</p>
    {:else}
      <p>🔋 Akku: Nicht verfügbar</p>
    {/if}
  </div>

  {#if needsOrientationPermission && !orientationPermissionGranted}
    <button class="permission-button" on:click={requestOrientationPermission}>
      Kompass aktivieren
    </button>
  {/if}
</div>
