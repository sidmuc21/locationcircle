<script>
  import { onMount, onDestroy } from 'svelte';
  import {
    isPointWithinRadius,
    getGreatCircleBearing,
    getDistance
  } from 'geolib';

  let userCoords = { latitude: null, longitude: null };
  let inRange = false;
  let bearingToTarget = 0;
  let distanceToTarget = null;

  let alpha = 0; // Compass direction of the phone
  let heading = 0; // Adjusted direction to face target

  const targetCoords = {
    latitude: 42.05913363079434,
    longitude: 19.51725368912721
  };

  let needsOrientationPermission = false;
  let orientationPermissionGranted = false;

  function startWatching() {
    if (!navigator.geolocation) {
      alert('Geolocation not supported');
      return;
    }

    navigator.geolocation.watchPosition(
      (pos) => {
        userCoords.latitude = pos.coords.latitude;
        userCoords.longitude = pos.coords.longitude;

        if (userCoords.latitude != null && userCoords.longitude != null) {
          inRange = isPointWithinRadius(userCoords, targetCoords, 5);
          distanceToTarget = Math.round(getDistance(userCoords, targetCoords));
          bearingToTarget = Math.round(getGreatCircleBearing(userCoords, targetCoords));
          heading = (bearingToTarget - alpha + 360) % 360;
        }
      },
      (err) => {
        console.error('Error getting location', err);
      },
      {
        enableHighAccuracy: true,
        maximumAge: 1000,
        timeout: 5000
      }
    );
  }

  function handleOrientation(event) {
    if (typeof event.alpha === 'number') {
      alpha = event.alpha;
      heading = (bearingToTarget - alpha + 360) % 360;
    }
  }

  function requestOrientationAccess() {
    if (typeof DeviceOrientationEvent !== 'undefined' && typeof DeviceOrientationEvent.requestPermission === 'function') {
      DeviceOrientationEvent.requestPermission()
        .then(state => {
          if (state === 'granted') {
            window.addEventListener('deviceorientation', handleOrientation, true);
            orientationPermissionGranted = true;
            needsOrientationPermission = false;
          } else {
            alert('Orientation permission denied');
          }
        })
        .catch(err => {
          console.error('Orientation permission error:', err);
        });
    }
  }

  onMount(() => {
    startWatching();

    if (typeof DeviceOrientationEvent !== 'undefined' && typeof DeviceOrientationEvent.requestPermission === 'function') {
      // iOS requires user interaction to request permission
      needsOrientationPermission = true;
    } else {
      // Android and others
      window.addEventListener('deviceorientation', handleOrientation, true);
      orientationPermissionGranted = true;
    }
  });

  onDestroy(() => {
    window.removeEventListener('deviceorientation', handleOrientation);
  });
</script>

<div class="container">
  <div class="circle" class:green={inRange}></div>

  {#if !inRange}
    <div
      class="arrow"
      style="transform: rotate({heading}deg);"
    ></div>
  {/if}

  <div class="info">
    <p>📍 Lat: {userCoords.latitude}</p>
    <p>📍 Lon: {userCoords.longitude}</p>
    <p>📏 Distance: {distanceToTarget} meters</p>
    <p>🧭 Bearing to target: {bearingToTarget}°</p>
    <p>📱 Alpha (compass): {Math.round(alpha)}°</p>
    <p>➡️ Adjusted Heading: {Math.round(heading)}°</p>
  </div>

  {#if needsOrientationPermission && !orientationPermissionGranted}
    <button class="permission-button" on:click={requestOrientationAccess}>
      Enable Compass
    </button>
  {/if}
</div>

<style>
  .container {
    position: relative;
    width: 100dvw;
    height: 100dvh;
    display: flex;
    justify-content: center;
    align-items: center;
    overflow: hidden;
    font-family: sans-serif;
    background: #f9f9f9;
  }

  .circle {
    width: 30dvw;
    height: 30dvw;
    max-width: 60dvh;
    max-height: 60dvh;
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
    top: 5dvh;
    left: 50%;
    transform-origin: bottom center;
    width: 0;
    height: 0;
    margin-left: -2dvw;
    border-left: 2dvw solid transparent;
    border-right: 2dvw solid transparent;
    border-bottom: 6dvh solid black;
    z-index: 2;
    transition: transform 0.2s linear;
  }

  .info {
    position: absolute;
    bottom: 2dvh;
    left: 2dvw;
    background: rgba(255, 255, 255, 0.9);
    padding: 1em;
    border-radius: 8px;
    font-size: 0.9rem;
    line-height: 1.5;
    color: #000;
    box-shadow: 0 0 10px rgba(0,0,0,0.1);
  }

  .permission-button {
    position: absolute;
    top: 2dvh;
    right: 2dvw;
    padding: 0.6em 1em;
    background: #007bff;
    color: white;
    border: none;
    border-radius: 6px;
    font-size: 1rem;
    cursor: pointer;
    z-index: 3;
    box-shadow: 0 2px 6px rgba(0,0,0,0.2);
  }

  .permission-button:hover {
    background: #0056b3;
  }
</style>
