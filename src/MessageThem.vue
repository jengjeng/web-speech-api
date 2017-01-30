<template>
  <div class="message message-them new">
    <figure class="avatar"><img :src="image"></figure>
    <span>{{ content }}</span>
    <div class="timestamp">{{ time.getHours() }}:{{ time.getMinutes() }}</div>
  </div>
</template>
<script>
  export default {
    props: ['image', 'content', 'time']
  }
</script>