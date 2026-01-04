<script setup>
import './assets/main.css'
import Header from './components/Header.vue'
import Configuration from './components/Configuration.vue'
import FAQ from './components/FAQ.vue'
import Footer from './components/Footer.vue'
</script>

<template>
  <div class="app-shell">
    <header class="app-header">
      <div class="container">
      </div>
      <Header addonName="Cinebye"
        addonSummary="Effortlessly get rid of Cinemeta, as much as you decide to." addonLogo="logo.png" />
    </header>
    <main class="app-main">
      <Configuration />
      <FAQ />
    </main>
    <footer class="app-footer">
      <Footer />
    </footer>
  </div>
</template>
