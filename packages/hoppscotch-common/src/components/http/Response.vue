<template>
  <div class="relative flex flex-1 flex-col">
    <HttpResponseMeta :response="doc.response" :is-embed="isEmbed" />
    <LensesResponseBodyRenderer
      v-if="!loading && hasResponse"
      v-model:document="doc"
      :is-editable="false"
      :tab-id="tabId"
      @save-as-example="saveAsExample"
    />
  </div>
  <HttpSaveResponseName
    v-model:response-name="responseName"
    :has-same-name-response="hasSameNameResponse"
    :show="showSaveResponseName"
    @submit="onSaveAsExample"
    @hide-modal="showSaveResponseName = false"
  />
</template>

<script setup lang="ts">
import { useVModel } from "@vueuse/core"
import { computed, ref } from "vue"
import { HoppRequestDocument } from "~/helpers/rest/document"
import { useResponseBody } from "@composables/lens-actions"
import { getStatusCodeReasonPhrase } from "~/helpers/utils/statusCodes"
import {
  HoppRESTRequestResponse,
  HoppRESTResponseOriginalRequest,
  makeHoppRESTResponseOriginalRequest,
} from "@hoppscotch/data"
import { editRESTRequest } from "~/newstore/collections"
import { useToast } from "@composables/toast"
import { useI18n } from "@composables/i18n"
import { runMutation } from "~/helpers/backend/GQLClient"
import { UpdateRequestDocument } from "~/helpers/backend/graphql"
import * as E from "fp-ts/Either"

const t = useI18n()
const toast = useToast()

const props = defineProps<{
  document: HoppRequestDocument
  tabId: string
  isEmbed: boolean
}>()

const emit = defineEmits<{
  (e: "update:tab", val: HoppRequestDocument): void
}>()

const doc = useVModel(props, "document", emit)

const hasResponse = computed(
  () =>
    doc.value.response?.type === "success" ||
    doc.value.response?.type === "fail"
)

const responseName = ref("")
const showSaveResponseName = ref(false)

const hasSameNameResponse = computed(() => {
  return responseName.value
    ? responseName.value in doc.value.request.responses
    : false
})

const loading = computed(() => doc.value.response?.type === "loading")

const saveAsExample = () => {
  showSaveResponseName.value = true
  responseName.value = doc.value.request.name
}

const onSaveAsExample = () => {
  const response = doc.value.response

  if (response && response.type === "success") {
    const { responseBodyText } = useResponseBody(response)

    const statusText = getStatusCodeReasonPhrase(
      response.statusCode,
      response.statusText
    )

    const {
      method,
      endpoint,
      headers,
      body,
      auth,
      params,
      name,
      requestVariables,
    } = response.req

    const originalRequest: HoppRESTResponseOriginalRequest =
      makeHoppRESTResponseOriginalRequest({
        method,
        endpoint,
        headers,
        body,
        auth,
        params,
        name,
        requestVariables,
      })

    const resName = responseName.value.trim()

    const responseObj: HoppRESTRequestResponse = {
      status: statusText,
      code: response.statusCode,
      headers: response.headers,
      body: responseBodyText.value,
      name: resName,
      originalRequest,
    }

    doc.value.request.responses = {
      ...doc.value.request.responses,
      [resName]: responseObj,
    }

    showSaveResponseName.value = false

    const saveCtx = doc.value.saveContext
    if (!saveCtx) return

    const req = doc.value.request

    if (saveCtx.originLocation === "user-collection") {
      try {
        editRESTRequest(saveCtx.folderPath, saveCtx.requestIndex, req)
        toast.success(`${t("response.saved")}`)
        responseName.value = ""
      } catch (e) {
        console.error(e)
        responseName.value = ""
      }
    } else {
      runMutation(UpdateRequestDocument, {
        requestID: saveCtx.requestID,
        data: {
          title: req.name,
          request: JSON.stringify(req),
        },
      })().then((result) => {
        if (E.isLeft(result)) {
          toast.error(`${t("profile.no_permission")}`)
          responseName.value = ""
        } else {
          doc.value.isDirty = false
          toast.success(`${t("request.saved")}`)
          responseName.value = ""
        }
      })
    }
  }
}
</script>
