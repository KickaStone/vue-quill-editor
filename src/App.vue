<script setup>
import EditorDialog from "@/components/EditorDialog.vue";
function onEditor(){
  EditorDialog.methods.openEditDialog(true, "create");
}

</script>

<template>
  <body>
    <EditorDialog ref="quill-editor"/>
    <el-button @click="onEditor">Save</el-button>
  </body>
</template>

<style scoped>
header {
  line-height: 1.5;
}

.logo {
  display: block;
  margin: 0 auto 2rem;
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
    place-items: flex-start;
    flex-wrap: wrap;
  }
}
</style>
