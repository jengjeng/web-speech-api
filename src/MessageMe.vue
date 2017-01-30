<template>
  <div class="message message-personal new">
    <span>{{ content }}</span>
  </div>
</template>
<script>
  export default {
    props: ['content']
  }
</script>