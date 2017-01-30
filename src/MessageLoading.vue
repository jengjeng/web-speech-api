<template>
  <div class="message message-them loading new">
    <figure class="avatar"><img :src="image"></figure>
    <span></span>
  </div>
</template>
<script>
  export default {
    props: ['image']
  }
</script>