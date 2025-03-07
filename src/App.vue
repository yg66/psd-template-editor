<script setup>
import PsdEditor from "@/components/PsdEditor.vue";
</script>

<template>
  <header>
    <img alt="Vue logo" class="logo" src="./assets/logo.svg" width="125" height="125" />

    <div class="wrapper">
      <h1>PSD 编辑器</h1>
      <p>上传PSD文件，点击图层进行编辑</p>
    </div>
  </header>

  <main>
    <PsdEditor />
  </main>
</template>

<style scoped>
header {
  line-height: 1.5;
  text-align: center;
  margin-bottom: 2rem;
}

.logo {
  display: block;
  margin: 0 auto 1rem;
}

h1 {
  font-size: 1.5rem;
  margin-bottom: 0.5rem;
}

main {
  display: flex;
  flex-direction: column;
  flex: 1;
  height: calc(100vh - 200px);
  min-height: 500px;
}

@media (min-width: 1024px) {
  header {
    display: flex;
    place-items: center;
    padding-right: calc(var(--section-gap) / 2);
  }

  .logo {
    margin: 0 2rem 0 0;
  }

  header .wrapper {
    display: flex;
    flex-direction: column;
    place-items: flex-start;
  }
}
</style>
