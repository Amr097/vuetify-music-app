<template>
  <div class="col-span-1">
    <div class="bg-white rounded border border-gray-200 relative flex flex-col">
      <div class="px-6 pt-6 pb-5 font-bold border-b border-gray-200">
        <span class="card-title">Upload</span>
        <i
          for="song-upload"
          class="fas fa-upload float-right text-green-400 text-2xl cursor-pointer"
        ></i>
      </div>
      <div class="p-6">
        <!-- Upload Dropbox -->
        <div
          class="w-full px-10 py-20 rounded text-center cursor-pointer border border-dashed border-gray-400 text-gray-400 transition duration-500"
          :class="{ 'text-white bg-green-400 border-green-400 border-solid': is_dragover }"
          @drag.prevent.stop=""
          @dragstart.prevent.stop=""
          @dragend.prevent.stop=""
          @dragenter.prevent.stop="is_dragover = true"
          @dragleave.prevent.stop="is_dragover = false"
          @dragover.prevent.stop="is_dragover = true"
          @drop.prevent.stop="upload($event)"
        >
          <h5>Drop your files here</h5>
        </div>
        <input id="song-upload" type="file" multiple @change="upload($event)" class="mt-4" />
        <hr class="my-6" />
        <!-- Progess Bars -->
        <div class="mb-4" v-for="file in uploads" :key="file.name">
          <!-- File Name -->
          <div class="font-bold text-sm" :class="file.text_class">
            <i :class="file.icon"></i>
            {{ file.name }}
          </div>
          <div class="flex h-4 overflow-hidden bg-gray-200 rounded">
            <!-- Inner Progress Bar -->
            <div
              class="transition-all progress-bar"
              :class="file.variant"
              :style="{ width: file.current_progess + '%' }"
            ></div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { storage, auth, songsCollection } from '../services/firebase'
import { ref, uploadBytesResumable, getDownloadURL } from 'firebase/storage'
import { addDoc, getDoc } from 'firebase/firestore'

export default {
  name: 'Upload',
  data() {
    return {
      is_dragover: false,
      uploads: []
    }
  },
  props: {
    addSong: {
      type: Function,
      required: true
    }
  },
  methods: {
    upload(event) {
      this.is_dragover = false
      const files = event.dataTransfer ? [...event.dataTransfer.files] : [...event.target.files]
      files.forEach((file) => {
        if (file.type !== 'audio/mpeg') {
          console.log(file.type)
          return
        }

        const storageRef = ref(storage, 'music-main')
        const musicRef = ref(storageRef, `music/${file.name}`)
        const musicUploadTask = uploadBytesResumable(musicRef, file)

        const uploadIndex =
          this.uploads.push({
            musicUploadTask,
            current_progress: 0,
            name: file.name,
            variant: 'bg-blue-400',
            icon: 'fas fa-spinner fa-spin',
            text_class: ''
          }) - 1

        musicUploadTask.on(
          'state_changed',
          (snapshot) => {
            const progress = (snapshot.bytesTransferred / snapshot.totalBytes) * 100
            this.uploads[uploadIndex].current_progess = progress
          },
          (error) => {
            this.uploads[uploadIndex].variant = 'bg-red-400'
            this.uploads[uploadIndex].icon = 'fa-solid fa-x'
            this.uploads[uploadIndex].text_class = 'text-red-400'
            console.log(error)
          },
          async () => {
            const song = {
              uid: auth.currentUser.uid,
              display_name: auth.currentUser.displayName,
              original_name: musicUploadTask.snapshot.ref.name,
              modified_name: musicUploadTask.snapshot.ref.name,
              genre: '',
              comment_count: 0,
              url: ''
            }

            song.url = await getDownloadURL(musicRef)

            const newSong = await addDoc(songsCollection, song)
            const newSongSnapshot = await getDoc(newSong)

            this.uploads[uploadIndex].variant = 'bg-green-400'
            this.uploads[uploadIndex].icon = 'fas fa-check'
            this.uploads[uploadIndex].text_class = 'text-green-400'
            this.addSong(newSongSnapshot)
          }
        )
      })
    },
    cancelUploads() {
      this.uploads.forEach((upload) => {
        upload.musicUploadTask.cancel()
      })
    }
  },
  beforeUnmount() {
    this.cancelUploads()
  }
}
</script>

<style lang="scss" scoped></style>
