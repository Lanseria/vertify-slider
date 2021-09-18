# vertify-slider

此项目为 vitejs vuejs3 库, 一起构建的组件

This project is for the vitejs vuejs3 library, built together with the components

## npm / yarn 安装

```bash
npm i @lanseria/vertify-slider

yarn add @lanseria/vertify-slider
```

## 引入 @lanseria/vertify-slider

```js
import "@lanseria/vertify-slider/style.css";
```

```vue
<template>
  <pdf-editor
    src="http://183.131.134.242:1889/files/20210619/383518b085f9469eb9ec34594196f8c7.pdf"
  ></pdf-editor>
</template>

<script lang="ts">
import { defineComponent } from "vue";
import { PdfEditor } from "@lanseria/vertify-slider";
// import { PdfViewer } from "@lanseria/vertify-slider";

export default defineComponent({
  name: "App",
  components: {
    PdfEditor,
  },
});
</script>
```

## 具体 TODO
