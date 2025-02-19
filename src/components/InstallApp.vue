<template>
  <mwc-fab
    extended
    icon="add"
    label="Install new app"
    @click="selectWebHappFile()"
    style="
      margin: 16px;
      position: absolute;
      right: 0;
      bottom: 0;
      --mdc-theme-secondary: #4720e3;
    "
  ></mwc-fab>
  <mwc-dialog
    heading="Install App"
    :open="!!webAppBundlePath"
    scrimClickAction=""
    escapeKeyAction=""
  >
    <div class="row center" style="width: 512px" v-if="isLoadingFile">
      <mwc-circular-progress
        indeterminate
        style="margin-top: 80px; margin-bottom: 60px"
      ></mwc-circular-progress>
    </div>
    <div
      class="column"
      style="width: 512px"
      v-else-if="webAppBundlePath && appInfo"
    >
      <span style="margin-right: 8px"
        >Bundle: {{ pathToFilename(webAppBundlePath) }}</span
      >

      <div class="column" style="flex: 1">
        <mwc-textfield
          label="App Id"
          outlined
          ref="appIdField"
          class="row-item"
          @input="appId = $event.target.value"
          autoValidate
          required
          helper="Has to be unique"
          style="flex: 1"
        ></mwc-textfield>
        <mwc-textfield
          label="UID"
          outlined
          @input="uid = $event.target.value"
          helper="Change it to create a new network"
          class="row-item"
          style="flex: 1"
        ></mwc-textfield>

        <span
          class="row-item"
          style="font-size: 18px; font-weight: 400; color: black"
          >Dna Roles</span
        >
        <div
          v-for="(appRole, index) of appInfo.roles_to_create"
          :key="appRole.id"
          class="column"
          style="flex: 1; margin-top: 8px"
        >
          <span>#{{ index + 1 }} {{ appRole.id }}</span>
          <mwc-textarea
            style="flex: 1; margin-top: 4px"
            label="Membrane Proof"
            outlined
            helper="Check with the author if this is required"
            @input="membraneProofs[appRole.id] = $event.target.value"
          ></mwc-textarea>
        </div>
      </div>
    </div>

    <mwc-button
      v-if="isFileLoaded"
      label="Cancel"
      slot="secondaryAction"
      :disabled="installing"
      @click="cleanStateAndClose()"
    >
    </mwc-button>
    <mwc-button
      v-if="isFileLoaded"
      slot="primaryAction"
      :disabled="!isAppReadyToInstall || installing"
      @click="installApp()"
      :label="installing ? 'Installing...' : 'Install app'"
    >
    </mwc-button>
  </mwc-dialog>
  <mwc-snackbar leading :labelText="snackbarText" ref="snackbar"></mwc-snackbar>
</template>

<script lang="ts">
import { defineComponent } from "vue";
import AdminUI from "@holochain/admin-ui";
import { invoke } from "@tauri-apps/api/tauri";
import { open } from "@tauri-apps/api/dialog";
import type { TextField } from "@material/mwc-textfield";
import { InstalledAppInfo } from "@holochain/client";
import { WebAppInfo } from "../types";
import { toUint8Array } from "js-base64";

type Dictionary<T> = { [key: string]: T };

export default defineComponent({
  name: "InstallApp",
  data(): {
    installing: boolean;
    appId: string | undefined;
    webAppBundlePath: string | undefined;
    uid: string | undefined;
    membraneProofs: { [key: string]: string } | undefined;
    appInfo: WebAppInfo | undefined;
    snackbarText: string | undefined;
    isAppIdValid: boolean;
  } {
    return {
      installing: false,
      appId: undefined,
      webAppBundlePath: undefined,
      uid: undefined,
      membraneProofs: undefined,
      appInfo: undefined,
      snackbarText: undefined,
      isAppIdValid: true,
    };
  },
  computed: {
    isAppReadyToInstall() {
      if (!this.appId) return false;
      if (!this.webAppBundlePath) return false;
      if (!this.isAppIdValid) return false;
      return true;
    },
    isFileLoaded() {
      if (this.appInfo) return true;
      return false;
    },
    isLoadingFile() {
      if (this.webAppBundlePath && !this.appInfo) return true;
      return false;
    },
  },
  methods: {
    async selectWebHappFile() {
      this.webAppBundlePath = (await open({
        filters: [{ name: "webhapp", extensions: ["webhapp"] }],
      })) as string;

      if (!this.webAppBundlePath) return;

      this.membraneProofs = {};
      this.appInfo = (await invoke("get_web_app_info", {
        webAppBundlePath: this.webAppBundlePath,
      })) as WebAppInfo;
      this.appId = this.appInfo.app_name;

      this.$nextTick(() => {
        setTimeout(() => {
          const appIdField = this.$refs.appIdField as TextField;
          appIdField.validityTransform = (newValue: string, nativeValidity) => {
            if (newValue === "") {
              appIdField.setCustomValidity("App Id is required");
              this.isAppIdValid = false;

              return {
                valid: false,
              };
            }

            const alreadyInstalledAppIds =
              this.$store.state.admin.installedApps.appsInfo.map(
                (appInfo: InstalledAppInfo) => appInfo.installed_app_id
              );
            const isValid = !alreadyInstalledAppIds.includes(newValue);

            // changes to make to the native validity
            if (!isValid) {
              appIdField.setCustomValidity("App Id already exists");
            }
            this.isAppIdValid = isValid;
            return {
              valid: isValid,
            };
          };
          appIdField.value = this.appId as string;
          appIdField.reportValidity();
        });
      });
    },
    cleanStateAndClose() {
      this.webAppBundlePath = undefined;
      this.uid = undefined;
      this.membraneProofs = undefined;
      this.appInfo = undefined;
    },
    pathToFilename(path: string) {
      const components = path.split("/");
      return components[components.length - 1];
    },
    getEncodedMembraneProofs() {
      if (!this.membraneProofs) return {};

      const encodedMembraneProofs: Dictionary<Array<number>> = {};
      for (const dnaSlot of Object.keys(this.membraneProofs)) {
        encodedMembraneProofs[dnaSlot] = Array.from(
          toUint8Array(this.membraneProofs[dnaSlot])
        );
      }
      console.log(encodedMembraneProofs);
      return encodedMembraneProofs;
    },
    async installApp() {
      try {
        this.installing = true;
        await invoke("install_app", {
          appId: this.appId,
          webAppBundlePath: this.webAppBundlePath,
          membraneProofs: this.getEncodedMembraneProofs(),
          uid: this.uid,
        });

        this.installing = false;
        await this.$store.dispatch(
          `${AdminUI.ADMIN_UI_MODULE}/${AdminUI.ActionTypes.fetchInstalledApps}`
        );

        this.showMessage(`Installed ${this.appId}`);

        this.cleanStateAndClose();
      } catch (e) {
        this.installing = false;
        this.showMessage(JSON.stringify(e));
      }
    },
    showMessage(message: string) {
      this.snackbarText = message;
      (this.$refs as any).snackbar.show();
    },
  },
});
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
.app-row {
  margin-bottom: 16px;
}

.app-title {
  font-size: 24px;
}

.cell-row {
  margin-top: 8px;
}
.center {
  align-items: center;
  justify-content: center;
}

.row-item {
  margin-top: 16px;
}
</style>
