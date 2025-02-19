<template>
  <mwc-dialog
    :heading="!launchError() ? 'Factory Reset' : 'Launch Error'"
    ref="dialog"
    scrimClickAction=""
    escapeKeyAction=""
  >
    <div class="column">
      <span v-if="launchError()">There was an error launching Holochain:</span>
      <pre v-if="launchError()" style="word-wrap: break-word">{{
        $store.state.connectionStatus.error
      }}</pre>
      <span v-if="launchError()" style="margin-top: 8px">
        If you are upgrading from a previous version of Holochain, it is most
        likely that the new version is not compatible with the data that this
        computer has stored.
      </span>
      <span v-if="launchError()" style="margin-top: 8px">
        In the future, Holochain will include a smooth mechanism to upgrade from
        one version to the next. Unfortunately, at the moment there is no way of
        upgrade old Holochain apps to be compatible with newer versions of
        Holochain.
      </span>
      <span v-if="launchError()" style="margin-top: 8px">
        <b>If you don't want to lose old Holochain data,</b> uninstall this
        version of the Holochain Launcher and downgrade with the version that
        you were already using.
      </span>
      <span style="margin-top: 8px">
        <b>If you don't mind losing old Holochain data</b>, you can execute a
        factory reset.
        <b>
          This will uninstall all the Holochain apps that were installed in this
          computer, and also remove all previous stored data.
        </b>
      </span>
    </div>

    <mwc-button
      slot="secondaryAction"
      v-if="!launchError()"
      :disabled="executing"
      dialogAction="close"
      label="Cancel"
    ></mwc-button>
    <mwc-button
      slot="primaryAction"
      @click="executeFactoryReset()"
      :disabled="executing"
      :label="executing ? 'Executing...' : 'Execute Factory Reset'"
    ></mwc-button>
  </mwc-dialog>
  <mwc-snackbar leading :labelText="snackbarText" ref="snackbar"></mwc-snackbar>
</template>

<script lang="ts">
import { defineComponent } from "vue";
import { invoke } from "@tauri-apps/api/tauri";
import { getCurrent, WebviewWindow } from "@tauri-apps/api/window";
import type { Dialog } from "@material/mwc-dialog";

export default defineComponent({
  name: "FactoryReset",
  data(): {
    snackbarText: string | undefined;
    executing: boolean;
  } {
    return {
      snackbarText: undefined,
      executing: false,
    };
  },
  async mounted() {
    this.$nextTick(async () => {
      if (this.launchError()) {
        this.showDialog();
      } else {
        const current = await getCurrent().listen("request-factory-reset", () =>
          this.showDialog()
        );
      }
    });
  },
  methods: {
    showDialog() {
      (this.$refs.dialog as Dialog).show();
    },
    launchError() {
      return this.$store.state.connectionStatus.type === "Error";
    },
    async executeFactoryReset() {
      try {
        this.executing = true;
        await invoke("execute_factory_reset");
        this.executing = false;
        window.location.reload();
      } catch (e) {
        this.executing = false;
        const error = `Error executing the factory reset: ${JSON.stringify(e)}`;
        this.showMessage(error);
        await invoke("log", {
          log: error,
        });
      }
    },
    showMessage(message: string) {
      this.snackbarText = message;
      (this.$refs as any).snackbar.show();
    },
  },
});
</script>
