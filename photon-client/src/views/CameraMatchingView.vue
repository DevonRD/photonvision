<script setup lang="ts">
import { useCameraSettingsStore } from "@/stores/settings/CameraSettingsStore";
import { computed, inject, ref } from "vue";
import { useStateStore } from "@/stores/StateStore";
import {
  PlaceholderCameraSettings,
  PVCameraInfo,
  type PVCSICameraInfo,
  type PVFileCameraInfo,
  type PVUsbCameraInfo
} from "@/types/SettingTypes";
import { getResolutionString } from "@/lib/PhotonUtils";
import PvCameraInfoCard from "@/components/common/pv-camera-info-card.vue";
import axios from "axios";
import _ from "lodash";
import PvCameraMatchCard from "@/components/common/pv-camera-match-card.vue";

const formatUrl = (port) => `http://${inject("backendHostname")}:${port}/stream.mjpg`;
const host = inject<string>("backendHost");

const activateModule = (moduleUniqueName: string) => {
  const url = new URL(`http://${host}/api/utils/activateMatchedCamera`);
  url.searchParams.set("uniqueName", moduleUniqueName);

  fetch(url.toString(), {
    method: "POST"
  });
};
const activateCamera = (cameraInfo: PVCameraInfo) => {
  const url = new URL(`http://${host}/api/utils/assignUnmatchedCamera`);
  url.searchParams.set("cameraInfo", JSON.stringify(cameraInfo));

  fetch(url.toString(), {
    method: "POST"
  });
};
const deactivateCamera = (cameraUniqueName: string) => {
  const url = new URL(`http://${host}/api/utils/unassignCamera`);
  url.searchParams.set("uniqueName", cameraUniqueName);

  fetch(url.toString(), {
    method: "POST"
  });
};

const deleteThisCamera = (cameraName: string) => {
  const payload = {
    cameraUniqueName: cameraName
  };

  axios
    .post("/utils/nukeOneCamera", payload)
    .then(() => {
      useStateStore().showSnackbarMessage({
        message: "Camera deleted successfully",
        color: "success"
      });
    })
    .catch((error) => {
      if (error.response) {
        useStateStore().showSnackbarMessage({
          message: "The backend is unable to fulfil the request to delete this camera.",
          color: "error"
        });
      } else if (error.request) {
        useStateStore().showSnackbarMessage({
          message: "Error while trying to process the request! The backend didn't respond.",
          color: "error"
        });
      } else {
        useStateStore().showSnackbarMessage({
          message: "An error occurred while trying to process the request.",
          color: "error"
        });
      }
    });
};

const cameraInfoFor = (camera: PVCameraInfo): PVUsbCameraInfo | PVCSICameraInfo | PVFileCameraInfo | any => {
  if (camera.PVUsbCameraInfo) {
    return camera.PVUsbCameraInfo;
  }
  if (camera.PVCSICameraInfo) {
    return camera.PVCSICameraInfo;
  }
  if (camera.PVFileCameraInfo) {
    return camera.PVFileCameraInfo;
  }
  return {};
};

const uniquePathForCamera = (info: PVCameraInfo) => {
  if (info.PVUsbCameraInfo) {
    return info.PVUsbCameraInfo.uniquePath;
  }
  if (info.PVCSICameraInfo) {
    return info.PVCSICameraInfo.uniquePath;
  }
  if (info.PVFileCameraInfo) {
    return info.PVFileCameraInfo.uniquePath;
  }

  // TODO - wut
  return "";
};

/**
 * Find the PVCameraInfo currently occupying the same uniquepath as the the given module
 */
const getMatchedDevice = (info: PVCameraInfo | undefined): PVCameraInfo => {
  if (!info) {
    return {
      PVFileCameraInfo: {
        name: "!",
        path: "!",
        uniquePath: "!"
      },
      PVCSICameraInfo: undefined,
      PVUsbCameraInfo: undefined
    };
  }
  return (
    useStateStore().vsmState.allConnectedCameras.find(
      (it) => uniquePathForCamera(it) === uniquePathForCamera(info)
    ) || {
      PVFileCameraInfo: {
        name: "!",
        path: "!",
        uniquePath: "!"
      },
      PVCSICameraInfo: undefined,
      PVUsbCameraInfo: undefined
    }
  );
};

const unmatchedCameras = computed(() => {
  const activeVmPaths = useCameraSettingsStore().cameras.map((it) => uniquePathForCamera(it.matchedCameraInfo));
  const disabledVmPaths = useStateStore().vsmState.disabledConfigs.map((it) =>
    uniquePathForCamera(it.matchedCameraInfo)
  );

  return useStateStore().vsmState.allConnectedCameras.filter(
    (it) => !activeVmPaths.includes(uniquePathForCamera(it)) && !disabledVmPaths.includes(uniquePathForCamera(it))
  );
});

const activeVisionModules = computed(() =>
  useCameraSettingsStore().cameras.filter(
    (camera) => JSON.stringify(camera) !== JSON.stringify(PlaceholderCameraSettings)
  )
);
const disabledVisionModules = computed(() => useStateStore().vsmState.disabledConfigs);

const viewingDetails = ref(false);
const viewingCamera = ref<PVCameraInfo | null>(null);

const setCameraView = (camera: PVCameraInfo | null) => {
  viewingDetails.value = camera !== null;
  viewingCamera.value = camera;
};
</script>

<template>
  <div class="pa-5">
    <v-row>
      <!-- Active modules -->
      <v-col
        cols="12"
        sm="6"
        lg="4"
        v-for="(module, index) in activeVisionModules"
        :key="`enabled-${module.uniqueName}`"
      >
        <v-card dark color="primary">
          <v-card-title>{{ module.nickname }}</v-card-title>
          <v-card-subtitle v-if="_.isEqual(getMatchedDevice(module.matchedCameraInfo), module.matchedCameraInfo)"
            >Status: <span class="active-status">Active</span></v-card-subtitle
          >
          <v-card-subtitle v-else>Status: <span class="mismatch-status">Mismatch</span></v-card-subtitle>
          <v-card-text>
            <v-simple-table dark dense>
              <tbody>
                <tr>
                  <td>Streams:</td>
                  <td>
                    <a :href="formatUrl(module.stream.inputPort)" target="_blank" class="stream-link">
                      Input Stream
                    </a>
                    /
                    <a :href="formatUrl(module.stream.outputPort)" target="_blank" class="stream-link">
                      Output Stream
                    </a>
                  </td>
                </tr>
                <tr>
                  <td>Pipelines</td>
                  <td>{{ module.pipelineNicknames.join(", ") }}</td>
                </tr>
                <tr>
                  <td>Connected</td>
                  <td>{{ module.isConnected }}</td>
                </tr>
                <tr>
                  <td>Calibrations</td>
                  <td>
                    {{
                      module.completeCalibrations.map((it) => getResolutionString(it.resolution)).join(", ") ||
                      "Not calibrated"
                    }}
                  </td>
                </tr>
                <tr v-if="module.isConnected && useStateStore().backendResults[index]">
                  <td>Frames Processed</td>
                  <td>
                    {{ useStateStore().backendResults[index].sequenceID }} ({{
                      useStateStore().backendResults[index].fps
                    }}
                    FPS)
                  </td>
                </tr>
              </tbody>
            </v-simple-table>
            <photon-camera-stream
              class="mt-3"
              id="output-camera-stream"
              :camera-settings="module"
              stream-type="Processed"
              style="width: 100%; height: auto"
            />
          </v-card-text>
          <v-card-text class="pt-0">
            <v-row>
              <v-col cols="12" md="4" class="pr-md-0 pb-0 pb-md-3">
                <v-btn color="secondary" @click="setCameraView(module.matchedCameraInfo)" style="width: 100%">
                  <span>Details</span>
                </v-btn>
              </v-col>
              <v-col cols="6" md="5" class="pr-0">
                <v-btn
                  class="black--text"
                  @click="deactivateCamera(module.uniqueName)"
                  color="accent"
                  style="width: 100%"
                >
                  Deactivate
                </v-btn>
              </v-col>
              <v-col cols="6" md="3">
                <v-btn
                  class="black--text pa-0"
                  @click="deleteThisCamera(module.uniqueName)"
                  color="red"
                  style="width: 100%"
                >
                  <v-icon>mdi-trash-can-outline</v-icon>
                </v-btn>
              </v-col>
            </v-row>
          </v-card-text>
        </v-card>
      </v-col>

      <!-- Disabled modules -->
      <v-col cols="12" sm="6" lg="4" v-for="module in disabledVisionModules" :key="`disabled-${module.uniqueName}`">
        <v-card dark color="primary">
          <v-card-title>{{ module.nickname }}</v-card-title>
          <v-card-subtitle>Status: <span class="inactive-status">Deactivated</span></v-card-subtitle>
          <v-card-text>
            <v-simple-table dense>
              <tbody>
                <tr>
                  <td>Name</td>
                  <td>
                    {{ module.cameraQuirks.baseName }}
                  </td>
                </tr>
                <tr>
                  <td>Pipelines</td>
                  <td>{{ module.pipelineNicknames.join(", ") }}</td>
                </tr>
                <tr>
                  <td>Connected</td>
                  <td>{{ module.isConnected }}</td>
                </tr>
                <tr>
                  <td>Calibrations</td>
                  <td>
                    {{
                      module.calibrations.map((it2) => getResolutionString(it2.resolution)).join(", ") ||
                      "Not calibrated"
                    }}
                  </td>
                </tr>
              </tbody>
            </v-simple-table>
          </v-card-text>
          <v-card-text class="pt-0">
            <v-row>
              <v-col cols="12" md="4" class="pr-md-0 pb-0 pb-md-3">
                <v-btn color="secondary" @click="setCameraView(module.matchedCameraInfo)" style="width: 100%">
                  <span>Details</span>
                </v-btn>
              </v-col>
              <v-col cols="6" md="5" class="pr-0">
                <v-btn
                  class="black--text"
                  @click="activateModule(module.uniqueName)"
                  color="accent"
                  style="width: 100%"
                >
                  Activate
                </v-btn>
              </v-col>
              <v-col cols="6" md="3">
                <v-btn
                  class="black--text pa-0"
                  @click="deleteThisCamera(module.uniqueName)"
                  color="red"
                  style="width: 100%"
                >
                  <v-icon>mdi-trash-can-outline</v-icon>
                </v-btn>
              </v-col>
            </v-row>
          </v-card-text>
        </v-card>
      </v-col>

      <!-- Unassigned cameras -->
      <v-col cols="12" sm="6" lg="4" v-for="(camera, index) in unmatchedCameras" :key="index">
        <v-card dark color="primary">
          <v-card-title>
            <span v-if="camera.PVUsbCameraInfo">USB Camera:</span>
            <span v-else-if="camera.PVCSICameraInfo">CSI Camera:</span>
            <span v-else-if="camera.PVFileCameraInfo">File Camera:</span>
            <span v-else>Unknown Camera:</span>
            &nbsp;<span>{{ cameraInfoFor(camera)?.name ?? cameraInfoFor(camera)?.baseName }}</span>
          </v-card-title>
          <v-card-subtitle>Status: Unassigned</v-card-subtitle>
          <v-card-text>
            <span style="word-break: break-all">{{ cameraInfoFor(camera)?.path }}</span>
          </v-card-text>
          <v-card-text class="pt-0">
            <v-row>
              <v-col cols="6" class="pr-0">
                <v-btn color="secondary" @click="setCameraView(camera)" style="width: 100%">
                  <span>Details</span>
                </v-btn>
              </v-col>
              <v-col cols="6">
                <v-btn class="black--text" @click="activateCamera(camera)" color="accent" style="width: 100%">
                  Activate
                </v-btn>
              </v-col>
            </v-row>
          </v-card-text>
        </v-card>
      </v-col>

      <!-- Info card -->
      <v-col cols="12" sm="6" lg="4">
        <v-card
          dark
          flat
          class="pl-6 pr-6 d-flex flex-column justify-center"
          style="background-color: transparent; height: 100%"
        >
          <v-card-text class="d-flex flex-column align-center justify-center">
            <v-icon size="64" color="primary">mdi-plus</v-icon>
          </v-card-text>
          <v-card-title>Additional plugged in cameras will display here!</v-card-title>
        </v-card>
      </v-col>
    </v-row>

    <!-- Camera details modal -->
    <v-dialog v-model="viewingDetails">
      <v-card dark flat color="primary" v-if="viewingCamera !== null">
        <v-card-title class="d-flex justify-space-between">
          <span>{{ cameraInfoFor(viewingCamera)?.name ?? cameraInfoFor(viewingCamera)?.baseName }}</span>
          <v-btn text @click="setCameraView(null)">
            <v-icon>mdi-close-thick</v-icon>
          </v-btn>
        </v-card-title>
        <v-card-text v-if="!_.isEqual(getMatchedDevice(viewingCamera), viewingCamera)">
          <v-banner
            rounded
            color="red"
            text-color="white"
            icon="mdi-information-outline"
            class="mb-3"
          >
            It looks like a different camera may have been connected to this device! Compare the
            below information carefully.
          </v-banner>
          <PvCameraMatchCard :saved="viewingCamera" :current="getMatchedDevice(viewingCamera)"/>
        </v-card-text>
        <v-card-text v-else>
          <PvCameraInfoCard :camera="viewingCamera" />
        </v-card-text>
      </v-card>
    </v-dialog>
  </div>
</template>

<style scoped>
.v-data-table {
  background-color: #006492 !important;
}

.active-status {
  color: rgb(14, 240, 14);
  background-color: transparent;
  text-decoration: none;
}

.inactive-status {
  color: red;
  background-color: transparent;
  text-decoration: none;
}

a:hover {
  color: pink;
  background-color: transparent;
  text-decoration: underline;
}

a:active,
.stream-link,
.mismatch-status {
  color: yellow;
  background-color: transparent;
  text-decoration: none;
}
</style>
