<template>
    <div id="app">
        <Header/>
        <Canvas/>
    </div>
</template>

<script>
import Header from './components/Header.vue'
import Canvas from './components/Canvas.vue'

export default {
    name: 'App',
    components: {
        Header,
        Canvas
    }
}
</script>

<style lang="scss">
    @import url('https://fonts.googleapis.com/css2?family=Fira+Mono:wght@500&family=Roboto:wght@500&display=swap');
    @import "../node_modules/normalize.css/normalize.css";
    html, body {
        height:100%;
        overflow: hidden;
    }
    #app {
        font-family: 'Roboto', sans-serif;
        font-weight:500;
        font-size:16px;
        height: 100%;
        color: #303438;
    }
    .inline {
        display:inline-flex;
        align-items:center;
    }
</style>
