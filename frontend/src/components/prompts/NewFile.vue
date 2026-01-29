<template>
  <div class="card-title">
    <h2>{{ $t("prompts.newFile") }}</h2>
  </div>

  <div class="card-content">
    <p>{{ $t("prompts.newFileMessage") }}</p>
    <input class="input" aria-label="FileName Field" v-focus type="text" @keyup.enter="submit"
      v-model.trim="name" />
    
    <!-- File Type Selection -->
    <div class="file-type-selector">
      <p class="file-type-label">{{ $t("prompts.fileType") || "File Type" }}</p>
      <div class="file-type-buttons">
        <button
          v-for="type in fileTypes"
          :key="type.id"
          :class="['file-type-button', { active: selectedFileType === type.id }]"
          @click="selectFileType(type.id)"
          :title="type.label"
          :aria-label="type.label"
        >
          <i class="material-icons">{{ type.icon }}</i>
          <span>{{ type.label }}</span>
        </button>
      </div>
    </div>
  </div>

  <div class="card-action">
    <button class="button button--flat button--grey" @click="closeHovers" :aria-label="$t('general.cancel')"
      :title="$t('general.cancel')">
      {{ $t("general.cancel") }}
    </button>
    <button class="button button--flat" @click="submit" :aria-label="$t('general.create')"
      :title="$t('general.create')">
      {{ $t("general.create") }}
    </button>
  </div>
</template>
<script>
import { state } from "@/store";
import { filesApi, publicApi } from "@/api";
import { getters, mutations } from "@/store"; // Import your custom store
import { notify } from "@/notify";
import { url } from "@/utils";
export default {
  name: "new-file",
  data() {
    return {
      name: "",
       selectedFileType: null,
       fileTypes: [
         { id: "document", label: "Document", icon: "description", ext: ".docx" },
         { id: "spreadsheet", label: "Spreadsheet", icon: "table_chart", ext: ".xlsx" },
         { id: "presentation", label: "Presentation", icon: "slideshow", ext: ".pptx" },
         { id: "text", label: "Text", icon: "text_fields", ext: ".txt" },
         { id: "markdown", label: "Markdown", icon: "markdown", ext: ".md" },
       ],
    };
  },
  computed: {
    isFiles() {
      return getters.isFiles();
    },
    isListing() {
      return getters.isListing();
    },
    closeHovers() {
      return mutations.closeHovers;
    },
  },
  methods: {
     selectFileType(typeId) {
       // Toggle: if already selected, deselect; otherwise select
       this.selectedFileType = this.selectedFileType === typeId ? null : typeId;
     },
     getFileNameWithExtension() {
       if (!this.selectedFileType) {
         return this.name;
       }
       const fileType = this.fileTypes.find(ft => ft.id === this.selectedFileType);
       return this.name + (fileType ? fileType.ext : "");
     },
    async submit(event) {
      try {
        event.preventDefault();
        if (this.name === "") return;
         await this.createFile(false);
      } catch (error) {
        console.error(error);
      }
    },

    async createFile(overwrite = false) {
      try {
         const fileName = this.getFileNameWithExtension();
         const newPath = url.joinPath(state.req.path, fileName);
        const source = state.req.source;

        if (getters.isShare()) {
          await publicApi.post(state.shareInfo?.hash, newPath, "", overwrite);
          mutations.setReload(true);
          mutations.closeHovers();
          return;
        }
        await filesApi.post(source, newPath, "", overwrite);
        mutations.setReload(true);
        mutations.closeHovers();

        // Show success notification with "go to item" button
        const buttonAction = () => {
          url.goToItem(source, newPath, {});
        };
        const buttonProps = {
          icon: "insert_drive_file",
          buttons: [{
            label: this.$t("buttons.goToItem"),
            primary: true,
            action: buttonAction
          }]
        };
        notify.showSuccess(this.$t("prompts.newFileSuccess"), buttonProps);
      } catch (error) {
        if (error.message === "conflict") {
          // Show replace-rename prompt for file/folder conflicts
          mutations.showHover({
            name: "replace-rename",
            confirm: async (event, option) => {
              event.preventDefault();
              try {
                if (option === "overwrite") {
                  await this.createFile(true); // Retry with overwrite
                } else if (option === "rename") {
                  // Add a suffix to make the name unique (max 100 attempts)
                  const originalName = this.name;
                  const maxAttempts = 100;
                  let success = false;
                  for (let counter = 1; counter <= maxAttempts && !success; counter++) {
                    try {
                      const newName = counter === 1 ? `${originalName} (1)` : `${originalName} (${counter})`;
                         const finalName = this.selectedFileType
                           ? newName + (this.fileTypes.find(ft => ft.id === this.selectedFileType)?.ext || "")
                           : newName;
                         const newPath = url.joinPath(state.req.path, finalName);
                      const source = state.req.source;

                      if (getters.isShare()) {
                        await publicApi.post(state.shareInfo?.hash, newPath, "", false);
                        mutations.setReload(true);
                        mutations.closeHovers();
                        success = true;
                        return;
                      }
                      await filesApi.post(source, newPath, "", false);
                      mutations.setReload(true);
                      mutations.closeHovers();
                      success = true;

                      // Show success notification with "go to item" button
                      const buttonAction = () => {
                        url.goToItem(source, newPath, {});
                      };
                      const buttonProps = {
                        icon: "insert_drive_file",
                        buttons: [{
                          label: this.$t("buttons.goToItem"),
                          primary: true,
                          action: buttonAction
                        }]
                      };
                      notify.showSuccess(this.$t("prompts.newFileSuccess"), buttonProps);
                    } catch (renameError) {
                      if (renameError.message === "conflict") {
                        // Continue to next iteration
                        continue;
                      } else {
                        throw renameError;
                      }
                    }
                  }
                  if (!success) {
                    throw new Error("Could not find a unique name after " + maxAttempts + " attempts");
                  }
                }
              } catch (retryError) {
                notify.showError(retryError);
              }
            },
          });
        } else {
          throw error;
        }
      }
    },
  },
};
</script>

  <style scoped>
  .file-type-selector {
    margin-top: 1.5rem;
    padding-top: 1rem;
    border-top: 1px solid rgba(0, 0, 0, 0.1);
  }

  .file-type-label {
    font-size: 0.95rem;
    font-weight: 500;
    margin: 0 0 0.75rem 0;
    color: var(--text-color, #333);
  }

  .file-type-buttons {
    display: flex;
    flex-wrap: wrap;
    gap: 0.75rem;
  }

  .file-type-button {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    padding: 0.5rem 1rem;
    border: 2px solid rgba(0, 0, 0, 0.15);
    border-radius: 0.5rem;
    background-color: transparent;
    color: #666;
    cursor: pointer;
    font-size: 0.9rem;
    transition: all 0.2s ease;
    user-select: none;
  }

  .file-type-button:hover:not(.active) {
    border-color: rgba(0, 0, 0, 0.3);
    background-color: rgba(0, 0, 0, 0.05);
  }

  .file-type-button.active {
    border-color: var(--primaryColor);
    background-color: var(--primaryColor);
    color: white;
  }

  .file-type-button i {
    font-size: 1.2rem;
  }

  .file-type-button span {
    white-space: nowrap;
  }

  /* Dark mode support */
  @media (prefers-color-scheme: dark) {
    .file-type-label {
      color: #e0e0e0;
    }
  
    .file-type-button {
      border-color: rgba(255, 255, 255, 0.2);
      color: #aaa;
    }
  
    .file-type-button:hover:not(.active) {
      border-color: rgba(255, 255, 255, 0.4);
      background-color: rgba(255, 255, 255, 0.1);
    }
  }
  </style>
