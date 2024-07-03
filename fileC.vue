<script setup lang="ts">
import cloneDeep from 'lodash/cloneDeep'
import AutoComplete from 'primevue/autocomplete'
import Button from 'primevue/button'
import Dropdown from 'primevue/dropdown'
import Inplace from 'primevue/inplace'
import InputMask from 'primevue/inputmask'
import InputText from 'primevue/inputtext'
import SelectButton from 'primevue/selectbutton'
import { useForm } from 'vee-validate'
import { watchEffect, MaybeRefOrGetter, ref, toRefs, computed, watch } from 'vue'
import AddressForm from '@/src/components/common/AddressForm.vue'
import { useCommonData } from '@/src/composables/api/useCommonData'
import { useMailDomains } from '@/src/composables/useMailDomains'
import { Location } from '@/src/types/Common/index.interface'
import { PersonData } from '@/src/types/Person/index.interface'
import { setCursor } from '@/src/utils/setCursor'
import { changePastedPhone, personSchema } from '@/src/validation/yupSchemes'

interface Props {
  initialData?: PersonData;
}

const props = defineProps<Props>()

const personForm = ref<PersonData>({
  gender: 1,
  family_status: 1,
  rr_citizenship_id: 810,
  location: {},
  identity_doc: {
    type_id: 10,
    rr_type_id: 2,
    rr_category_id: 1
  }
} as PersonData)

const personFormIsValid = ref<boolean>(false)

const { initialData } = toRefs(props)

if (initialData?.value) {
  personForm.value = cloneDeep(initialData.value)
}

const gender = [
  {
    label: 'М',
    value: 1
  },
  {
    label: 'Ж',
    value: 2
  }
]
const status = [
  {
    female: 'Не замужем',
    male: 'Холост',
    value: 1
  },
  {
    female: 'Замужем',
    male: 'Женат',
    value: 2
  }
]

const isForeignPassport = computed(() => {
  return personForm.value.identity_doc.rr_type_id === 4
})

const { getRRDictionaries } = useCommonData()
const { getDadata } = useCommonData()
const { search, mailItems } = useMailDomains()

const getAddress = () => {
  const address = personForm.value.location.note
  const { data, onFetchResponse } = getDadata(address)

  onFetchResponse(() => {
    personForm.value.location = { ...data.value, note: data.value.result }
  })
}

const getStatus = (val: number) => {
  return personForm.value.gender === 1 ? val.male : val.female
}

// Валидация форм
const { defineComponentBinds, errors, meta, setValues } = useForm({
  validationSchema: personSchema(personForm)
})

// Устанавливает значения всех полей, запускает проверку измененных полей
setValues({
  snils: personForm.value.snils,
  inn: personForm.value.inn,
  email: personForm.value.email,
  phone: personForm.value.phone,
  birthDate: personForm.value.birth_date,
  issueDate: personForm.value.identity_doc.issue_date,
  surname: personForm.value.surname,
  name: personForm.value.name,
  series: personForm.value.identity_doc.series,
  number: personForm.value.identity_doc.number,
  issuer_code: personForm.value.identity_doc.issuer_code,
  issuer: personForm.value.identity_doc.issuer,
  birth_place: personForm.value.birth_place
}, false)

const snils = defineComponentBinds('snils' as MaybeRefOrGetter)
const inn = defineComponentBinds('inn' as MaybeRefOrGetter)
const email = defineComponentBinds('email' as MaybeRefOrGetter)
const phone = defineComponentBinds('phone' as MaybeRefOrGetter)
const birthDate = defineComponentBinds('birthDate' as MaybeRefOrGetter)
const issueDate = defineComponentBinds('issueDate' as MaybeRefOrGetter)
const surname = defineComponentBinds('surname' as MaybeRefOrGetter)
const name = defineComponentBinds('name' as MaybeRefOrGetter)
const series = defineComponentBinds('series' as MaybeRefOrGetter)
const number = defineComponentBinds('number' as MaybeRefOrGetter)
const issuerCode = defineComponentBinds('issuer_code' as MaybeRefOrGetter)
const issuer = defineComponentBinds('issuer' as MaybeRefOrGetter)
const birthPlace = defineComponentBinds('birth_place' as MaybeRefOrGetter)
const foreignPhone = defineComponentBinds('foreign_phone' as MaybeRefOrGetter)

const { data: citizenship } = getRRDictionaries('DCountry')
const { data: documentType } = getRRDictionaries(1)

watchEffect(() => {
  personFormIsValid.value = meta.value.valid && meta.value.dirty
})

watch(() => personForm.value.identity_doc.rr_type_id, () => {
  if (personForm.value.identity_doc.rr_type_id === 4) {
    personForm.value.identity_doc.series = '0000'
    personForm.value.identity_doc.issuer_code = '000-000'
  } else {
    personForm.value.identity_doc.series = ''
    personForm.value.identity_doc.issuer_code = ''
  }
})

const onChangeAddress = ({ key, value }: {key: keyof Location; value: string | number}) => {
  (personForm.value.location[key] as string | number) = value
}

defineExpose({ personForm, personFormIsValid })
</script>

<style scoped lang="scss">
.container {
  @apply w-full;
}

.person-gender-options{
  :deep( .p-button ){
    @apply w-1/2;
  }
}

</style>