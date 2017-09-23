<template>
    <div id="app">
        <div class="container">
            <Changelog></Changelog>
        </div>
    </div>
</template>

<script>
    import Changelog from './components/Changelog'

    export default {
        name: 'app',
        components: {
            Changelog
        }
    }
</script>