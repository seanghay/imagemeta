<template>
  <div class="text-center p-6 max-w-1024px mx-auto">
    <h1 class="font-bold text-3xl">ImageMeta</h1>
    <p class="text-gray-500 my-2 text-sm">A tool for inspecting image file</p>
    <button @click="$refs.imageInput.value = null; $refs.imageInput.click();"
      class="bg-gray-500 cursor-pointer text-white rounded border-none px-3 py-2">Choose a file</button>
    <div v-if="fileRef" class="my-6">
      <div class="bg-white max-w-720px mx-auto border border-gray-300 p-2" v-if="imageRef">
        <img v-if="imageRef" v-bind="imageRef" alt="" style="display: block; width: 100%; height: auto;">
      </div>


      <div v-if="fileInfoRef" class="mt-4 text-left max-w-740px mx-auto">
        <h1 class="font-bold text-2xl mb-4">File Info</h1>
        <div :class="[index === 0 && 'border-t border-t-gray-300']"
          class="bg-white border-b border-l border-r px-4 border-b-gray-300 border-l-gray-300 border-r-gray-300 py-2"
          v-for="(item, index) in fileInfoRef" :key="item.key">
          <h1 class="font-semibold text-sm text-gray-800">{{ item.key }}</h1>
          <p class="text-wrap break-all text-sm mt-1 text-gray-600 ">{{ item.value }}</p>
        </div>

        <label class="mt-4 block text-sm font-bold" for="compress-format">Compression format</label>
        <select v-model="compressionFormatRef" id="compress-format"
          class="block text-sm my-2 border-gray-200 px-3 py-2">
          <option :value="item" v-for="item in supportedCompressionTypes" :key="item">{{ item }}</option>
        </select>

        <label class="mt-4 block text-sm font-bold" for="maxSize">Max Size</label>
        <input v-model="imageMaxSizeRef" min="0"
          class="block my-2 border text-sm border-gray-200 px-3 py-2 outline-none" type="number" name="maxSize"
          id="maxSize">



        <button :disabled="isExporting" @click="compressFile"
          class="bg-orange-500 hover:bg-orange-600 transition cursor-pointer text-white rounded border-none px-3 py-2 mt-2">
          <span v-if="isExporting">Loading...</span>
          <span v-else>Save</span>
        </button>
      </div>

      <div v-if="exifRef" class="mt-4 text-left max-w-740px mx-auto">
        <h1 class="font-bold text-2xl mb-4">EXIF</h1>
        <div :class="[index === 0 && 'border-t border-t-gray-300']"
          class="bg-white border-b border-l border-r px-4 border-b-gray-300 border-l-gray-300 border-r-gray-300 py-2"
          v-for="(item, index) in exifRef" :key="item.key">
          <h1 class="font-semibold text-sm text-gray-800">{{ item.key }}</h1>
          <p class="text-wrap break-all text-sm mt-1 text-gray-600 ">{{ item.value.description }}</p>
        </div>
      </div>
    </div>
    
    <div class="my-4 pt-4 ">
      <a class="inline-block text-sm text-gray-500" href="https://github.com/seanghay/imagemeta">View on GitHub</a>
      <br>
      <span  class="inline-block mt-2 text-sm text-gray-500">
        Made with ❤️ by
        <a class="text-gray-500" href="https://github.com/seanghay/imagemeta">@seanghay</a>
      </span>
    </div>


  </div>
  <input @change="onImageFileChange" ref="imageInput" type="file" accept="image/*" hidden>
</template>

<script setup>
import ExifReader from 'exifreader'
import { computed, ref, watch } from 'vue'
import prettyBytes from 'pretty-bytes'
import imageCompression from 'browser-image-compression'
import { saveAs } from 'file-saver';

const fileRef = ref(null)
const imageRef = ref(null)
const exifRef = ref(null);
const fileInfoRef = ref(null);
const imageMaxSizeRef = ref(0)
const compressionFormatRef = ref("image/jpeg")
const isExporting = ref(false)

const mimeTypeMap = {
  "image/png": "png",
  "image/jpeg": "jpg",
  "image/webp": "webp",
  "image/gif": "gif",
  "image/avif": "avif"
}

const supportedCompressionTypes = ref(Object.keys(mimeTypeMap))
const compressFormatExt = computed(() => mimeTypeMap[compressionFormatRef.value])


watch(fileRef, () => {
  if (!fileRef.value) return;
  readImageBase64();
  readExif();
  readFileInfo()
})

async function readFileInfo() {
  const file = fileRef.value;

  fileInfoRef.value = [
    { key: "File Name", value: file.name },
    { key: "Mime Type", value: file.type },
    { key: "File Size", value: prettyBytes(file.size) },
  ]
}

function readImageBase64() {
  imageRef.value = null;
  const reader = new FileReader();
  reader.onload = () => {
    const image = new Image();
    image.onload = () => {
      const { width, height } = image
      imageRef.value = {
        width,
        height,
        src: reader.result,
      }

      imageMaxSizeRef.value = width;
    }
    image.src = reader.result;
  }
  reader.readAsDataURL(fileRef.value);
}


async function readExif() {
  const file = fileRef.value;
  const data = await ExifReader.load(file);
  const entries = Object.entries(data);
  exifRef.value = entries.map(([key, value]) => ({ key, value }))
}

function onImageFileChange(e) {
  fileRef.value = e.target.files[0];
}

async function compressFile() {
  const file = fileRef.value
  if (!file) return;
  isExporting.value = true
  const maxSize = isNaN(imageMaxSizeRef.value) ? Number.POSITIVE_INFINITY : parseInt(imageMaxSizeRef.value);

  const result = await imageCompression(file, {
    maxWidthOrHeight: maxSize,
    fileType: compressionFormatRef.value,
  });

  const output = prettyBytes(result.size);
  console.log(`size=${output}`)
  const filename = file.name.split('.').slice(0, -1).join('.')
  await saveAs(result, filename + `.${compressFormatExt.value}`)
  isExporting.value = false
}
</script>