<script>
  import { onMount } from 'svelte';
  import {
    isPointWithinRadius,
    getGreatCircleBearing,
    getDistance
  } from 'geolib';

  let userCoords = { latitude: null, longitude: null };
  let inRange = false;
  let bearingToTarget = 0;
  let distanceToTarget = null;

  const targetCoords = {
    latitude: 42.05913363079434,
    longitude: 19.51725368912721
  };

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

          distanceToTarget = Math.round(getDistance(userCoords, targetCoords)); // meters, rounded
          bearingToTarget = Math.round(getGreatCircleBearing(userCoords, targetCoords)); // degrees, rounded
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

  onMount(() => {
    startWatching();
  });
</script>

<!-- UI -->
<div class="container">
  <!-- Circle changes color based on proximity -->
  <div class="circle" class:green={inRange}></div>

  <!-- Arrow points to target only when out of range -->
  {#if !inRange}
    <div
      class="arrow"
      style="transform: rotate({bearingToTarget}deg);"
    ></div>
  {/if}

  <!-- Coordinates and Distance Info -->
  <div class="info">
    <p>📍 Lat: {userCoords.latitude}</p>
    <p>📍 Lon: {userCoords.longitude}</p>
    <p>📏 Distance: {distanceToTarget} meters</p>
    <p>🧭 Direction to target: {bearingToTarget}°</p>
  </div>
</div>

<style>
  .container {
    position: relative;
    width: 100vw;
    height: 100vh;
    display: flex;
    justify-content: center;
    align-items: center;
    overflow: hidden;
    font-family: sans-serif;
    background: black;
  }

  .circle {
    width: 30vw;
    height: 30vw;
    max-width: 60vh;
    max-height: 60vh;
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
    top: 5vh;
    left: 50%;
    transform-origin: bottom center;
    width: 0;
    height: 0;
    margin-left: -2vw;
    border-left: 2vw solid transparent;
    border-right: 2vw solid transparent;
    border-bottom: 6vh solid black;
    z-index: 2;
    transition: transform 0.2s linear;
  }

  .info {
    position: absolute;
    bottom: 2vh;
    left: 2vw;
    background: rgba(255, 255, 255, 0.8);
    padding: 1em;
    border-radius: 8px;
    font-size: 0.9rem;
    line-height: 1.5;
    color: #000;
  }
</style>
