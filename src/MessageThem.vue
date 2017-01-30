<template>
  <div class="message message-them new">
    <figure class="avatar"><img :src="image"></figure>
    <span>{{ content }}</span>
    <div class="timestamp">{{ time.toTimeString().split(' ')[0].slice(0, 5) }}</div>
  </div>
</template>
<script>
  export default {
    props: ['image', 'content', 'time']
  }
</script>