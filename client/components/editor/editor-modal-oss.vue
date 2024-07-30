<template lang='pug'>
v-card.editor-modal-media.animated.fadeInLeft(
  flat,
  tile,
  :class="`is-editor-` + editorKey"
)
  v-container.pa-3(grid-list-lg, fluid)
    v-layout(row, wrap)
      v-flex(xs6, offset-xs4)
        v-card.radius-7.animated.fadeInRight.wait-p3s(
          :light="!$vuetify.theme.dark",
          :dark="$vuetify.theme.dark"
        )
          v-card-text
            .d-flex
              v-toolbar.radius-7(
                :color="$vuetify.theme.dark ? `teal` : `teal lighten-5`",
                dense,
                flat,
                height="44"
              )
                v-icon.mr-3(:color="$vuetify.theme.dark ? `white` : `teal`") mdi-cloud-upload
                .body-2(
                  :class="$vuetify.theme.dark ? `white--text` : `teal--text`"
                ) {{ $t("editor:assets.uploadAssets") }}
              v-btn.my-0.ml-3.mr-0.radius-7(
                outlined,
                large,
                color="teal",
                @click="browse",
                v-if="$vuetify.breakpoint.mdAndUp"
              )
                v-icon(left) mdi-plus-box-multiple
                span(
                  :class="$vuetify.theme.dark ? `teal--text text--lighten-3` : ``"
                ) {{ $t("common:actions.browse") }}
            file-pond.mt-3(
              name="mediaUpload",
              ref="pond",
              :label-idle="$t(`editor:assets.uploadAssetsDropZone`)",
              allow-multiple="false",
              :files="files",
              max-files="1",
              :server="filePondServerOpts",
              :instant-upload="false",
              :allow-revert="false",
              @processfile="onFileProcessed"
            )
          v-divider
          v-card-actions.pa-3
            .caption.grey--text.text-darken-2 Max 1 files, 20 MB
            v-spacer
            v-btn.px-4(color="teal", dark, @click="upload") {{ $t("common:actions.upload") }}

        v-card.mt-3.radius-7.animated.fadeInRight.wait-p4s(
          :light="!$vuetify.theme.dark",
          :dark="$vuetify.theme.dark"
        )
          v-card-text.pb-0
            v-toolbar.radius-7(
              :color="$vuetify.theme.dark ? `teal` : `teal lighten-5`",
              dense,
              flat
            )
              v-icon.mr-3(:color="$vuetify.theme.dark ? `white` : `teal`") mdi-format-align-top
              .body-2(
                :class="$vuetify.theme.dark ? `white--text` : `teal--text`"
              ) {{ $t("editor:assets.imageAlign") }}
            v-select.mt-3(
              v-model="imageAlignment",
              :items="imageAlignments",
              outlined,
              single-line,
              color="teal",
              placeholder="None"
            )
</template>

<script>
import _ from "lodash";
import moment from "moment";
import { get, sync } from "vuex-pathify";
import vueFilePond from "vue-filepond";
import "filepond/dist/filepond.min.css";

import listAssetQuery from "gql/editor/editor-media-query-list.gql";
import listFolderAssetQuery from "gql/editor/editor-media-query-folder-list.gql";

const FilePond = vueFilePond();
const localeSegmentRegex = /^[A-Z]{2}(-[A-Z]{2})?$/i;
const disallowedFolderChars = /[A-Z()=.!@#$%?&*+`~<>,;:\\/[\]¬{| ]/;

export default {
  components: {
    FilePond,
  },
  props: {
    value: {
      type: Boolean,
      default: false,
    },
  },
  data() {
    return {
      sts: {},
      client: null,
      folders: [],
      files: [],
      assets: [],
      pagination: 1,
      remoteImageUrl: "",
      imageAlignments: [
        { text: "None", value: "" },
        { text: "Left", value: "left" },
        { text: "Centered", value: "center" },
        { text: "Right", value: "right" },
        { text: "Absolute Top Right", value: "abstopright" },
      ],
      imageAlignment: "",
      loading: false,
      newFolderDialog: false,
      newFolderName: "",
      newFolderLoading: false,
      previewDialog: false,
      renameDialog: false,
      renameAssetName: "",
      renameAssetLoading: false,
      deleteDialog: false,
      deleteAssetLoading: false,
    };
  },
  computed: {
    isShown: {
      get() {
        return this.value;
      },
      set(val) {
        this.$emit("input", val);
      },
    },
    editorKey: get("editor/editorKey"),
    activeModal: sync("editor/activeModal"),
    folderTree: get("editor/media@folderTree"),
    currentFolderId: sync("editor/media@currentFolderId"),
    currentFileId: sync("editor/media@currentFileId"),
    pageTotal() {
      if (!this.assets) {
        return 0;
      }

      return Math.ceil(this.assets.length / 15);
    },
    headers() {
      return _.compact([
        this.$vuetify.breakpoint.smAndUp && {
          text: this.$t("editor:assets.headerId"),
          value: "id",
          width: 80,
        },
        { text: this.$t("editor:assets.headerFilename"), value: "filename" },
        this.$vuetify.breakpoint.lgAndUp && {
          text: this.$t("editor:assets.headerType"),
          value: "ext",
          width: 90,
        },
        this.$vuetify.breakpoint.mdAndUp && {
          text: this.$t("editor:assets.headerFileSize"),
          value: "fileSize",
          width: 110,
        },
        this.$vuetify.breakpoint.mdAndUp && {
          text: this.$t("editor:assets.headerAdded"),
          value: "createdAt",
          width: 175,
        },
        this.$vuetify.breakpoint.smAndUp && {
          text: this.$t("editor:assets.headerActions"),
          value: "",
          width: 80,
          sortable: false,
          align: "right",
        },
      ]);
    },
    isFolderNameValid() {
      return (
        this.newFolderName.length > 1 &&
        !localeSegmentRegex.test(this.newFolderName) &&
        !disallowedFolderChars.test(this.newFolderName)
      );
    },
    currentAsset() {
      return _.find(this.assets, ["id", this.currentFileId]) || {};
    },
    filePondServerOpts() {
      // const jwtToken = Cookies.get("jwt");
      return {
        process: (
          fieldName,
          file,
          metadata,
          load,
          error,
          progress,
          abort,
          transfer,
          options
        ) => {
          const fileNames = file.name.split(".");
          const name =
            this.sts.dir + this.guid() + "." + fileNames[fileNames.length - 1];

          //start upload
          let abortCheckpoint;
          this.client
            .multipartUpload(name, file, {
              progress: (p, cpt, res) => {
                // 获取分片上传进度、断点和返回值。
                // console.log(p, cpt, res);
                abortCheckpoint = cpt;
                progress(true, p, 1);
              },
              parallel: 4, // 设置并发上传的分片数量。
              partSize: 1024 * 1024, // 设置分片大小。默认值为1 MB，最小值为100 KB。
            })
            .then(() => {
              const url = `https://${this.sts.host}/${name}`;
              load(url);
            })
            .catch((err) => {
              if (err.name === "abort") {
                console.log("error: ", err.message);
              }
              error(err.message);
            });

          return {
            abort: () => {
              // This function is entered if the user has tapped the cancel button
              // request.abort();
              this.client.abortMultipartUpload(
                abortCheckpoint.name,
                abortCheckpoint.uploadId
              );
              // Let FilePond know the request has been cancelled
              abort();
            },
          };
        },
      };
    },
  },
  watch: {
    newFolderDialog(newValue, oldValue) {
      if (newValue) {
        this.$nextTick(() => {
          this.$refs.folderNameIpt.focus();
        });
      }
    },
  },
  filters: {
    prettyBytes(num) {
      if (typeof num !== "number" || isNaN(num)) {
        throw new TypeError("Expected a number");
      }

      let exponent;
      let unit;
      let neg = num < 0;
      let units = ["B", "kB", "MB", "GB", "TB", "PB", "EB", "ZB", "YB"];

      if (neg) {
        num = -num;
      }
      if (num < 1) {
        return (neg ? "-" : "") + num + " B";
      }
      exponent = Math.min(
        Math.floor(Math.log(num) / Math.log(1000)),
        units.length - 1
      );
      num = (num / Math.pow(1000, exponent)).toFixed(2) * 1;
      unit = units[exponent];

      return (neg ? "-" : "") + num + " " + unit;
    },
  },
  methods: {
    async refresh() {
      await this.$apollo.queries.assets.refetch();
      this.$store.commit("showNotification", {
        message: this.$t("editor:assets.refreshSuccess"),
        style: "success",
        icon: "check",
      });
    },
    insert(text, path) {
      this.$root.$emit("editorInsert", {
        kind: "IMAGE",
        path,
        text,
        align: this.imageAlignment,
      });
      this.activeModal = "";
    },
    browse() {
      this.$refs.pond.browse();
    },
    async upload() {
      const files = this.$refs.pond.getFiles();
      if (files.length < 1) {
        return this.$store.commit("showNotification", {
          message: this.$t("editor:assets.noUploadError"),
          style: "warning",
          icon: "warning",
        });
      }
      this.sts = await this.getSTSToken();

      function handleToken(data) {
        return {
          accessKeyId: data.AccessKeyId,
          accessKeySecret: data.AccessKeySecret,
          stsToken: data.SecurityToken,
        };
      }

      this.client = new OSS({
        region: this.sts.region,
        bucket: this.sts.bucket,
        ...handleToken(this.sts),
      });

      await this.$refs.pond.processFiles();
    },
    guid() {
      function S4() {
        return (((1 + Math.random()) * 0x10000) | 0).toString(16).substring(1);
      }
      return (
        moment().format("YYYY-MM-DD") +
        "/" +
        S4() +
        S4() +
        "-" +
        S4() +
        "-" +
        S4() +
        "-" +
        S4() +
        "-" +
        S4()
      );
    },

    async getSTSToken() {
      return new Promise((resolve, reject) => {
        OSS.urllib.request(
          "https://mmdoc.t5.wmeimob.cn/oss/wiki/sts",
          { method: "POST" },
          (err, response) => {
            if (err) {
              reject(err);
              return alert(err);
            }
            try {
              const result = JSON.parse(response);
              resolve(result);
            } catch (e) {
              errmsg = "parse sts response info error: " + e.message;
              reject(errmsg);
              return alert(errmsg);
            }
          }
        );
      });
    },
    async onFileProcessed(err, file) {
      if (err) {
        return this.$store.commit("showNotification", {
          message: this.$t("editor:assets.uploadFailed"),
          style: "error",
          icon: "error",
        });
      }
      _.delay(() => {
        this.$refs.pond.removeFile(file.id);
        const namePaths = file.serverId.split("/");
        this.insert(namePaths[namePaths.length - 1], file.serverId);
      }, 300);

      // await this.$apollo.queries.assets.refetch();
    },
    downFolder(folder) {
      this.$store.commit("editor/pushMediaFolderTree", folder);
      this.currentFolderId = folder.id;
      this.currentFileId = null;
    },
    upFolder() {
      this.$store.commit("editor/popMediaFolderTree");
      const parentFolder = _.last(this.folderTree);
      this.currentFolderId = parentFolder ? parentFolder.id : 0;
      this.currentFileId = null;
    },
    cancel() {
      this.activeModal = "";
    },
  },
  apollo: {
    folders: {
      query: listFolderAssetQuery,
      variables() {
        return {
          parentFolderId: this.currentFolderId,
        };
      },
      fetchPolicy: "network-only",
      update: (data) => data.assets.folders,
      watchLoading(isLoading) {
        this.$store.commit(
          `loading${isLoading ? "Start" : "Stop"}`,
          "editor-media-folders-list-refresh"
        );
      },
    },
    assets: {
      query: listAssetQuery,
      variables() {
        return {
          folderId: this.currentFolderId,
          kind: "ALL",
        };
      },
      throttle: 1000,
      fetchPolicy: "network-only",
      update: (data) => data.assets.list,
      watchLoading(isLoading) {
        this.loading = isLoading;
        this.$store.commit(
          `loading${isLoading ? "Start" : "Stop"}`,
          "editor-media-list-refresh"
        );
      },
    },
  },
};
</script>

<style lang='scss'>
.editor-modal-media {
  position: fixed !important;
  top: 112px;
  left: 64px;
  z-index: 10;
  width: calc(100vw - 64px - 17px);
  height: calc(100vh - 112px - 24px);
  background-color: rgba(darken(mc("grey", "900"), 3%), 0.9) !important;
  overflow: auto;

  @include until($tablet) {
    left: 40px;
    width: calc(100vw - 40px);
    height: calc(100vh - 112px - 24px);
  }

  &.is-editor-ckeditor {
    top: 64px;
    left: 0;
    width: 100%;
    height: calc(100vh - 64px - 26px);

    @include until($tablet) {
      top: 56px;
      left: 0;
      width: 100%;
      height: calc(100vh - 56px - 24px);
    }
  }

  &.is-editor-code {
    top: 64px;
    height: calc(100vh - 64px - 26px);

    @include until($tablet) {
      top: 56px;
      height: calc(100vh - 56px - 24px);
    }
  }

  &.is-editor-common {
    top: 64px;
    left: 0;
    width: 100%;
    height: calc(100vh - 64px);

    @include until($tablet) {
      top: 56px;
      left: 0;
      width: 100%;
      height: calc(100vh - 56px);
    }
  }

  .filepond--root {
    margin-bottom: 0;
  }

  .filepond--drop-label {
    cursor: pointer;

    > label {
      cursor: pointer;
    }
  }

  .filepond--file-action-button.filepond--action-process-item {
    display: none;
  }

  .v-btn--icon {
    padding: 0 20px;
  }
}
</style>
