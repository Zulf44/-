<template>
    <div>
      <Button
        @click="visible = true"
        class="m-1"
        severity="warning"
        label="Письмо в техподдержку SignMe"
      />
      <Dialog
        v-model:visible="visible"
        header="Письмо в техподдержку SignMe"
        class="w-[30vw] min-w-[400px]"
      >
        <div
          v-for="category in supportDicts"
          :key="category.id"
          class="flex align-items-center"
        >
          <div class="mt-2">
            <RadioButton
              @update:model-value="(newValue: number) => refreshMailText(newValue)"
              v-model="selectedCategory"
              :input-id="category.id"
              :value="category.id"
            />
            <label :for="category.id" class="ml-2">{{ category.subject }}</label>
          </div>
        </div>
        <div class="flex justify-center">
          <Textarea
            style=""
            class="mt-5 "
            v-model="mailText"
            rows="10"
            cols="80"
            placeholder="Поле для свободного описания проблемы"
          />
        </div>
  
        <FileUpload
          class="mt-5"
          mode="basic"
          accept=".jpg, .jpeg, .png, .pdf"
          :max-file-size="30000000"
          choose-label="Выбрать файл"
          :auto="true"
          custom-upload
          @uploader="event => uploadDocuments(event.files)"
        />
        <div class="w-full flex justify-end">
          <Button
            @click="sendEmail"
            class="m-1"
            severity="warning"
            label="Отправить"
          />
        </div>
      </Dialog>
    </div>
  </template>
  
  <script setup lang="ts">
  import { serialize } from 'object-to-formdata'
  import Button from 'primevue/button'
  import Dialog from 'primevue/dialog'
  import FileUpload from 'primevue/fileupload'
  import RadioButton from 'primevue/radiobutton'
  import Textarea from 'primevue/textarea'
  import { ref, watch } from 'vue'
  import { useSignMe } from '@/src/composables/api/useSignMe.js'
  import { SignMeStatus } from '@/src/types/Person/signMe.interface.js'
  
  interface Props {
    personId?: number;
  }
  
  const props = defineProps<Props>()
  
  const { signMeSupportDicts, signMeSendSupportEmail } = useSignMe()
  const supportDicts = signMeSupportDicts()
  const visible = ref(false)
  const mailText = ref('')
  const selectedCategory = ref('')
  let files: File | File[] | null = null
  
  const refreshMailText = (newValue: number) => {
    if (supportDicts.value) {
      const category = supportDicts.value.find((item: SignMeStatus) => item.id === +newValue)
      mailText.value = ''
      if (category.message) {
        mailText.value = category.message
      }
    }
  }
  const uploadDocuments = (file: File | File[]) => {
    if (Array.isArray(file)) {
      // Если передан массив файлов
      files = file[0]
    } else {
      // Если передан один файл
      files = file
    }
  }
  
  const sendEmail = () => {
    const payload = {
      person_id: props.personId, text: mailText.value, files
    }
    const formData = serialize(payload)
    const { onFetchResponse } = signMeSendSupportEmail(formData)
    mailText.value = ''
    selectedCategory.value = ''
    files = null
    visible.value = false
    // хук не отрабатывает потому что сервер шлет пустую строку, разобраться позже
    onFetchResponse(() => {
      visible.value = false
    })
  }
  
  </script>
  