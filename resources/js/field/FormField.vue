<template>
  <default-field :field="field" :show-help-text="field && !!field.helpText">
    <template slot="field">

      <template v-if="field.value && field.display == 'image'">
        <div class="card relative card relative border-lg border-50 overflow-hidden px-0 py-0 max-w-xs mb-2">
          <template v-if="field.type == 'image'">
            <ImageDetail class="block w-full" :file="field" :css="''"></ImageDetail>
          </template>

          <template v-else>
            <object class="no-preview" v-html="field.image">
            </object>
          </template>
        </div>
      </template>

      <modal-filemanager
          ref="filemanager"
          :resource="resourceName"
          :name="field.attribute"
          :home="field.home"
          :active="openModal"
          :currentPath="currentPath"
          :defaultFolder="defaultFolder"
          :filter="field.filterBy"
          :buttons="field.buttons"
          v-on:open-modal="openModalCreateFolder"
          v-on:close-modal="closeFilemanagerModal"
          v-on:update-current-path="updateCurrentPath"
          v-on:showInfoItem="showInfoItem"
          v-on:uploadFiles="uploadFiles"
          :value="value"
      />

      <DetailPopup
          ref="detailPopup"
          :info="info"
          :active="activeInfo"
          :popup="true"
          :buttons="field.buttons"
          v-on:closePreview="closePreview"
          v-on:refresh="refreshCurrent"
          v-on:selectFile="setValue"
          v-on:rename="fileRenamed"
      />

      <create-folder ref="createFolderModal" :active="showCreateFolder" :current="currentPath"
                     v-on:closeCreateFolderModal="closeModalCreateFolder" v-on:refresh="refreshCurrent"/>

      <UploadProgress ref="uploader" :current="currentPath" :visibility="field.visibility" :rules="field.upload_rules"
                      v-on:removeFile="removeFileFromUpload"></UploadProgress>

      <file-select :id="field.name" :field="field" :is-readonly="field.readonly" :css="errorClasses" v-model="value"
                   v-on:open-modal="openFilemanagerModal"></file-select>

      <p class="mt-3 flex items-center text-sm" v-if="value">
        <button type="button" class="cursor-pointer dim btn btn-link text-primary inline-flex items-center"
                @click="openRemoveModal">
          <icon type="delete" view-box="0 0 20 20" width="16" height="16"/>
          <span class="class ml-2 mt-1">
                        {{ __('Delete') }}
                    </span>
        </button>
      </p>

      <confirm-modal-remove-file
          :active="removeModalOpen"
          @confirm="removeFile"
          @close="closeRemoveModal"></confirm-modal-remove-file>

      <p v-if="hasError" class="my-2 text-danger">
        {{ firstError }}
      </p>
    </template>
  </default-field>
</template>

<script>
import {FormField, HandlesValidationErrors} from 'laravel-nova';
import FileSelect from './custom/FileSelect';
import ModalFileManager from '../components/ModalFileManager';
import CreateFolderModal from '../components/CreateFolderModal';
import DetailPopup from '../components/DetailPopup';
import UploadProgress from '../components/UploadProgress';
import ConfirmModalRemoveFile from '../components/ConfirmModalRemoveFile';
import ImageDetail from '../modules/Image';

import ConfirmModalDelete from '../components/ConfirmModalDelete';
import RenameModal from '../components/RenameModal';

import api from '../api';

export default {
  mixins: [FormField, HandlesValidationErrors],

  props: ['resourceName', 'resourceId', 'field'],

  components: {
    'file-select': FileSelect,
    'modal-filemanager': ModalFileManager,
    'create-folder': CreateFolderModal,
    DetailPopup: DetailPopup,
    UploadProgress: UploadProgress,
    'confirm-modal-remove-file': ConfirmModalRemoveFile,
    ImageDetail: ImageDetail,
    'rename-modal': RenameModal,
    'confirm-modal-delete': ConfirmModalDelete,
  },

  data: () => ({
    openModal: false,
    showCreateFolder: false,
    defaultFolder: null,
    currentPath: '/',

    //modalFile
    info: {},
    activeInfo: false,
    popupDetailsLoaded: false,

    //uploader
    filesToUpload: {},
    uploadType: null,
    folderUploadedName: null,

    removeModalOpen: false,
  }),

  methods: {
    openModalCreateFolder() {
      this.showCreateFolder = true;
    },
    closeModalCreateFolder() {
      this.showCreateFolder = false;
    },

    refreshCurrent() {
      this.$refs.filemanager.getData(this.currentPath);
    },

    openFilemanagerModal() {
      this.setCurrentPath();
      this.openModal = true;
    },

    closeFilemanagerModal() {
      this.openModal = false;
    },

    updateCurrentPath(val) {
      this.currentPath = val;
    },

    showInfoItem(item) {
      this.activeInfo = true;
      this.info = item;
    },

    closePreview() {
      this.info = {};
      this.activeInfo = false;
      this.popupDetailsLoaded = false;
    },

    uploadFiles(files, type, firstFolderName) {
      this.filesToUpload = files;
      this.uploadType = type;
      this.folderUploadedName = firstFolderName;
      this.$refs.uploader.startUploadingFiles(files, type);
    },

    removeFileFromUpload(uploadedFileId) {
      let index = this.filesToUpload.map(item => item.id).indexOf(uploadedFileId);

      this.$delete(this.filesToUpload, index);
      if (this.filesToUpload.length === 0) {
        if (this.uploadType == 'folders') {
          this.callFolderEvent(this.folderUploadedName);
        }

        this.folderUploadedName = null;
        this.uploadType = null;

        this.refreshCurrent();
      }
    },

    setCurrentPath() {
      if (this.field.folder != null) {
        this.defaultFolder = this.field.folder;
        this.currentPath = this.field.folder;
      } else {
        this.defaultFolder = '/';
        this.currentPath = '/';
      }
    },

    removeFile() {
      this.field.value = null;
      this.value = '';
      this.removeModalOpen = false;
    },

    fileRenamed(item) {
      this.info = item;
    },

    openRemoveModal() {
      this.removeModalOpen = true;
    },

    closeRemoveModal() {
      this.removeModalOpen = false;
    },

    callFolderEvent(path) {
      api.eventFolderUploaded(this.currentPathFolder + '/' + path);
    },

    /*
     * Set the initial, internal value for the field.
     */
    setInitialValue() {
      this.value = this.field.value || '';
    },

    /**
     * Fill the given FormData object with the field's internal value.
     */
    fill(formData) {
      formData.append(this.field.attribute, this.value || '');
    },

    /**
     * Update the field's internal value.
     */
    setValue(file) {
      this.value = file.path;
      this.closeFilemanagerModal();
    },
  },

  created() {
    this.setCurrentPath();
  },
};
</script>
