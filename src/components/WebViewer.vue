<template>
  <div>
    <label for="file_upload">Choose A file</label>
    <input type="file" id="file_upload" name="file_upload" accept=".pdf" />
    <button id="button">Save</button>
    <div ref="viewer"></div>
  </div>
</template>

<script>
import WebViewer from '@pdftron/pdfjs-express';

export default {
  name: 'WebViewer',
  data() {
    return {
      filename: null,
    };
  },
  props: {
    path: String,
    url: String,
  },
  mounted: function() {
    WebViewer(
      {
        path: this.path,
        initialDoc: this.url, // replace with your own PDF file
        disabledElements: [
          'zoomOutButton',
          'zoomInButton',
          'eraserToolButton',
          'textToolGroupButton',
          'toolsButton',
          'searchButton',
          'miscToolGroupButton',
          'signatureToolButton',
          'viewControlsButton',
          'panToolButton',
          'leftPanelButton',
        ],
      },
      this.$refs.viewer
    ).then((instance) => {
      const input = document.getElementById('file_upload');
      input.addEventListener('change', () => {
        // Get the file from the input
        const file = input.files[0];
        instance.loadDocument(file, { filename: file.name });
        this.filename = file.name;
      });

      const { docViewer, annotManager } = instance;
      const button = document.getElementById('button');

      button.addEventListener('click', async () => {
        const xfdf = await annotManager.exportAnnotations({ links: false, widgets: false });
        const fileData = await docViewer.getDocument().getFileData({});
        const blob = new Blob([fileData], { type: 'application/pdf' });

        const data = new FormData();
        data.append('xfdf', xfdf);
        data.append('file', blob);

        try {
          // Process the file
          const response = await fetch('https://api.pdfjs.express/xfdf/set', {
            method: 'post',
            body: data,
          }).then((resp) => resp.json());

          const { url, key } = response;
          // Download the file
          await fetch(url, {
            headers: {
              Authorization: key,
            },
          }).then(async (resp) => {
            const result = await resp.blob()
            console.log({result})
            // create link to download blob
            const _OBJECT_URL = URL.createObjectURL(result);
            var link = document.createElement('a');
            document.body.appendChild(link);
            link.download = `edited-${this.filename}`;
            link.href = _OBJECT_URL;
            link.click();
            // remove objectUrl and link from DOM
            link.remove();
            setTimeout(function() {
              window.URL.revokeObjectURL(_OBJECT_URL);
            }, 60 * 1000);
          });
        } catch (error) {
          console.log({ error });
        }
      });

      docViewer.on('documentLoaded', () => {
        console.log({ docViewer });
      });
    });
  },
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
div {
  width: 100%;
  height: 100vh;
}
</style>
