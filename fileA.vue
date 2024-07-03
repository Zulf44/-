<template>
    <div class="relative">
      <div class="card">
        <TimeLineSignatureStatus :status-stage="statusStage" :events="events" />
      </div>
      <div class="flex flex-wrap w-full justify-evenly mt-10">
        <Card v-if="!personIsRegistered" class="card-item">
          <template #title>
            <span>
              Регистрация
            </span>
          </template>
          <template #content>
            <div
              class="before-register p-2 mt-4"
            >
              <CourierDeliveryDocuments />
              <Button
                @click="register"
                class="mt-5"
                label="Зарегистрировать"
              />
            </div>
          </template>
        </Card>
        <Card v-if="personIsRegistered" class="card-item">
          <template #title>
            <span>
              Действия
            </span>
          </template>
          <template #content>
            <div
              class="before-register p-2 mt-4"
            >
              <Button
                class="m-1"
                severity="danger"
                label="Удалить IDP"
                @click="delUser"
              />
              <Button
                class="m-1"
                label="Обновить данные"
                @click="getRegStatus"
              />
            </div>
          </template>
        </Card>
        <Card v-if="personIsRegistered && statusStage === 2" class="card-item">
          <template #title>
            <span>
              Идентификации клиента
            </span>
          </template>
          <template #content>
            <FileUpload
              :choose-label="regDocList.title ? regDocList.title : 'Скан паспорта'"
              class="m-1"
              mode="basic"
              accept=".jpg, .jpeg, .png, .pdf"
              :max-file-size="30000000"
              custom-upload
              :disabled="!!regDocList.title"
              @uploader="event => uploadDocs(event.files, regDocList.id)"
            />
            <Button
              class="mt-5"
              label="Подтвердить идентификацию"
              :disabled="!regDocList.path"
              @click="changeState(workflowID, 'identify')"
            />
          </template>
        </Card>
        <Card v-if="personIsRegistered && statusStage >= 3" class="card-item">
          <template #title>
            <span>
              Загрузка документов для КЭП
            </span>
          </template>
          <template #content>
            <FileUpload
              v-for="item in docListCert"
              :key="item.id"
              :choose-label="item.document_title"
              class="m-1"
              mode="basic"
              accept=".jpg, .jpeg, .png, .pdf"
              :max-file-size="30000000"
              custom-upload
              :disabled="!!item.title"
              @uploader="event => uploadDocs(event.files, item.id, '-cert')"
            />
            <Button
              class="mt-5"
              label="Подтвердить"
              @click="changeState(childrenWorkflowID, 'received')"
            />
          </template>
        </Card>
      </div>
      <div v-if="loader" class="w-full h-full absolute top-0 flex justify-center items-center bg-white">
        <ProgressSpinner />
      </div>
    </div>
  </template>
  
  <script setup lang="ts">
  import { serialize } from 'object-to-formdata'
  import Button from 'primevue/button'
  import Card from 'primevue/card'
  import FileUpload from 'primevue/fileupload'
  import ProgressSpinner from 'primevue/progressspinner'
  import { ref, onMounted } from 'vue'
  import CourierDeliveryDocuments from '@/src/components/Person/ElectronicSignatute/CourierDeliveryDocuments.vue'
  import TimeLineSignatureStatus from '@/src/components/Person/ElectronicSignatute/TimeLineSignatureStatus.vue'
  import { useIDPoint } from '@/src/composables/api/useIDPoint'
  import { RegConfirmationDocument, RegRequestStatus, WorkflowStatus } from '@/src/types/Person/idpoint.interface'
  
  interface Props {
    personId?: number;
  }
  
  const { idPointRegistration, idPointDelPerson, idPointSearchRegRequest, idPointWorkflowInfo, idPointRegDocList, idPointChangeState, idPointDocListCert, idPointRegUploadDocs } = useIDPoint()
  
  const props = defineProps<Props>()
  
  const regDocList = ref<RegConfirmationDocument>({
    created: '',
    document: undefined,
    id: 0,
    path: '',
    state: '',
    template_text_url: '',
    template_url: '',
    title: '',
    type: '',
    updated: ''
  })
  
  const identificationWorkflowInfo = ref<WorkflowStatus>({
    consumer: undefined,
    consumer_role_type: '',
    id: 0,
    scenario: undefined,
    state: '',
    state_display: '',
    type: ''
  })
  
  const regStatus = ref<RegRequestStatus>({ children: [], consumer: undefined, created: '', id: 0, state: '', updated: '' })
  const loader = ref(false)
  const workflowID = ref(null)
  const childrenWorkflowID = ref(null)
  const events = ['Регистрация', 'Активация в приложении', 'Идентификации клиента', 'Загрузка документов для КЭП', 'Завершен' ]
  const statusStage = ref(0)
  const personIsRegistered = ref(false)
  const docListCert = ref([])
  
  const register = () => {
    const { onFetchResponse } = idPointRegistration(props.personId)
    loader.value = true
    onFetchResponse(() => {
      getRegStatus()
    })
  }
  
  const delUser = () => {
    const { onFetchResponse } = idPointDelPerson(props.personId)
    loader.value = true
    onFetchResponse(() => {
      statusStage.value = 0
      personIsRegistered.value = false
      getRegStatus()
    })
  }
  
  const changeState = (workflowID: number, state: string) => {
    const { onFetchResponse } = idPointChangeState(workflowID, state)
    loader.value = true
    onFetchResponse(() => {
      getRegStatus()
    })
  }
  
  const getRegStatus = () => {
    const { data, onFetchResponse } = idPointSearchRegRequest(props.personId)
    loader.value = true
    onFetchResponse(() => {
      if (data.value.result) {
        regStatus.value = data.value.result
  
        if (regStatus.value.state === 'complete') {
          statusStage.value = 3
        } else if (regStatus.value.state === 'wait-consumer-identification') {
          statusStage.value = 2
        } else statusStage.value = 1
  
        workflowID.value = data.value.result.id
        personIsRegistered.value = true
  
        if (data.value.result.children.length) {
          childrenWorkflowID.value = data.value.result.children[0]
          getWorkflowInfo(childrenWorkflowID.value, true)
        } else {
          getWorkflowInfo(workflowID.value)
          getIdPointRegDocList()
        }
  
        if (statusStage.value === 3 && data.value.result.children.length) {
          getIdPointDocListCert()
        }
      }
      loader.value = false
    })
  }
  
  const getWorkflowInfo = (workflowID: number, isChildren = false) => {
    const { data, onFetchResponse } = idPointWorkflowInfo(workflowID)
  
    onFetchResponse(() => {
      if (data.value.result.length !== 0) {
        identificationWorkflowInfo.value = data.value.result
        if (isChildren && identificationWorkflowInfo.value.state === 'complete') {
          statusStage.value = 5
        }
      }
    })
  }
  
  const getIdPointRegDocList = () => {
    const { data, onFetchResponse } = idPointRegDocList(workflowID.value)
  
    onFetchResponse(() => {
      if (data.value.result) {
        regDocList.value = data.value.result.confirmation_documents[0]
      }
    })
  }
  
  const getIdPointDocListCert = () => {
    const { data, onFetchResponse } = idPointDocListCert(childrenWorkflowID.value)
  
    onFetchResponse(() => {
      if (data.value.result) {
        docListCert.value = data.value.result.scans
      }
      if (docListCert.value.every(doc => doc.title !== '')) {
        statusStage.value = 4
      }
    })
  }
  
  const uploadDocs = (event: File | File[], docID: number, reqType = '') => {
    const file = Array.isArray(event) ? event[0] : event
    const payload = {
      person_id: props.personId, file
    }
    const formData = serialize(payload)
    idPointRegUploadDocs(formData, docID, reqType)
    getRegStatus()
  }
  
  onMounted(() => {
    getRegStatus()
  })
  
  </script>
  