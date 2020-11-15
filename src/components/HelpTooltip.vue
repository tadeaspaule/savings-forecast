<template>
  <q-icon name="help" style="display:inline;">
    <q-tooltip content-style="background:#eee;color:black;font-size: 14px;width:300px;">
      {{text}}
    </q-tooltip>
  </q-icon>
</template>
<script>
export default {
  props: {
    text: String
  }
}
</script>
