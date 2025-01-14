<template>
  <div class="iframe-container">
    <iframe
      ref="iframe"
      sandbox="allow-modals allow-forms allow-pointer-lock allow-popups allow-same-origin allow-scripts"
      scrolling="yes"
      frameborder="0"
    />
  </div>
</template>

<script setup lang="ts">
import { inject, onMounted, ref, watch } from 'vue';
import type { Store } from './store';

const defineImport = 'https://unpkg.com/es-module-shims@0.10.1/dist/es-module-shims.min.js';
const defineDep: { [key in string]: string } = {
  vue: 'https://unpkg.com/@vue/runtime-dom@3.2.31/dist/runtime-dom.esm-browser.js',
};
interface globalProps {
  store?: Store;
  readonly?: boolean;
  autoResize?: boolean;
  clearConsole?: boolean;
  depLibs?: { name: string; url: string }[];
  depCss: string[];
  layout?: 'horizontal' | 'vertical';
}

const store = inject<Store>('store');

const globalProp = inject<globalProps>('globalProps');
const iframe = ref<HTMLIFrameElement>().value

onMounted(()=>setIframe());

watch(
  () => store!.state.file.compiled.js,
  () => {
    setHTML(iframe!);
  },
);

function setIframe() {
  if (!iframe || !iframe.contentWindow) { return; }
  const iframeDocument = iframe.contentWindow.document;
  const stylesTags = globalProp?.depCss.map((style: string) => `<link rel='stylesheet' href='${style}' />`);
  
  globalProp?.depLibs?.forEach((lib) => {
    defineDep[lib.name] = lib.url;
  });

  const html = `
          <!DOCTYPE html>
            <html>
              <head>
                <script async src='${defineImport}'><\/script>
                <script type="importmap" crossorigin="anonymous">{"imports":${JSON.stringify(defineDep)}}<\/script>
                ${stylesTags!.join('\n')}
              </head>
              <body id='body'>
                <div>
                  <pre id='error' style='color: red'></pre>
                </div>
                <div id='app'></div>
              </body>
          </html>`;
  iframeDocument.open();
  iframeDocument.write(html);
  iframeDocument.close();
  iframe.addEventListener('load', () => {
    setHTML(iframe);
  });
}

function getScript(script?: string) {
  return ` 
    ${script}
  import { createApp as _createApp } from 'vue'
      const AppComponent =__sfc__
      AppComponent.name = 'Repl'
      const app = (window.__app__ = _createApp(AppComponent))
      app.config.unwrapInjec tedRef = true
      app.config.errorHandler = (e) => console.error(e)
      app.mount('#app')
      `;
}
// 设置html
function setHTML(iframe: HTMLIFrameElement) {
  if (!iframe || !iframe.contentWindow) { return; }
  const iframeDocument = iframe!.contentWindow!.document;

  if (iframeDocument) {
    const elBox = iframeDocument.querySelector('#body');
    if (elBox) {
      const fragment = iframeDocument.createDocumentFragment();
      const newStyle = iframeDocument.createElement('style');
      // 创建样式
      if (store?.state.file.compiled.css) {
        newStyle.type = 'text/css';
        newStyle.innerHTML = store?.state.file.compiled.css;
      }

      // 创建元素
      const elApp = iframeDocument.createElement('div');
      elApp.setAttribute('id', 'app');
      // 创建js
      const newScript = iframeDocument.createElement('script');
      newScript.type = 'module';
      newScript.innerHTML = getScript(store?.state.file.compiled.js);

      // 重置 html
      elBox.innerHTML = '';

      fragment.append(newScript);
      fragment.append(elApp);
      fragment.append(newStyle);

      elBox.append(fragment);
    }
  }
}
</script>

<style scoped>
.iframe-container,
.iframe-container :deep(iframe) {
  width: 100%;
  height: 100%;
  border: none;
}
</style>
