<template>
  <api-test-ui :api-test="apiTest" :perform-check="performCheck" />

  <check-submit-tx
    :log-id-suffix="logIdSuffix"
    :serialized-tx="serializedTx"
    :submit-tx="submitTx"
    v-if="serializedTx && submitTx"
  />
</template>

<script lang="ts">
import { computed, defineComponent, reactive, ref } from "vue";

import { addLogImportant, LogLevel, useApiLog } from "../useApiLog";

import {
  ApiTest,
  ApiTestStatus,
  createApiTest,
  setApiTestStatus,
} from "../lib/ApiTest";

import { isArray, returnsPromise } from "../lib/utils";
import {  MakeTxForSigning } from "../lib/utilsCbor";

import { addApiTest } from "../lib/ApiTestSuite";

import ApiTestUi from "./apiTestUi.vue";
import CheckSubmitTx from "./submitTx.vue";

export default defineComponent({
  name: "checkSignTx",

  props: {
    getUtxos: { type: Function, required: true },
    signTx: { type: Function, required: true },
    submitTx: { type: Function, required: true },
    logIdSuffix: { type: String, required: true },
  },

  components: {
    ApiTestUi,
    CheckSubmitTx,
  },

  setup(props) {
    const {
      getLog,
      clearLog,

      addLogSucceeded,
      addLogError,
    } = useApiLog();

    const apiTest: ApiTest = reactive<ApiTest>(
      createApiTest("signTx", "Check", ["- check signTx popup"]),
    );

    const logId = apiTest.label;
    const logs = getLog(logId);

    const showAllLogs = ref(false);

    const txBody = ref<string | null>(null);
    const witnesses = ref<string | null>(null);
    const serializedTx = ref<string | null>(null);

    const filteredLogs = computed(() => {
      return logs.filter(
        (item) =>
          item.level === LogLevel.error ||
          item.level <=
            (showAllLogs.value ? LogLevel.error : LogLevel.important),
      );
    });

    function resetStatus() {
      clearLog(logId);

      txBody.value = null;
      witnesses.value = null;
      serializedTx.value = null;

      setApiTestStatus(apiTest, ApiTestStatus.idle);
    }

    resetStatus();

    addApiTest(apiTest);

    function setApiTestFailed(msg: string) {
      addLogError(logId, "<b>" + msg + "</b>");
      setApiTestStatus(apiTest, ApiTestStatus.failed);
    }

    async function performCheck() {
      resetStatus();

      setApiTestStatus(apiTest, ApiTestStatus.running);

      try {
        let utxos: any = await returnsPromise(logId, "getUtxos", props.getUtxos);

        if (!isArray(utxos)) {
          return setApiTestFailed("getUtxos: return type not array");
        }

        const tx = MakeTxForSigning(utxos)


        addLogImportant(logId, "TX to sign: " + tx);

        let r: string = await returnsPromise(logId, "signTx", props.signTx, [
          tx, /*partial_sign=*/true
        ]);
        addLogImportant(logId, "Signed TX: " + r);

      } catch (e: any) {
        serializedTx.value = null;
        addLogError(logId, "signTx: error: " + JSON.stringify(e, null, 2));
        return setApiTestFailed(e.message);
      }

      setApiTestStatus(apiTest, ApiTestStatus.succeeded);
    }

    // performCheck()

    return {
      logs: filteredLogs,
      showAllLogs,

      apiTest,
      performCheck,

      txBody,
      witnesses,
      serializedTx,
    };
  },
});
</script>
