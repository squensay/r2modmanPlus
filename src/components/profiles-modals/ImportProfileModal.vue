<script lang="ts">
import { mixins } from "vue-class-component";
import { Component } from 'vue-property-decorator';

import ManagerInformation from "../../_managerinf/ManagerInformation";
import R2Error from "../../model/errors/R2Error";
import ExportFormat from "../../model/exports/ExportFormat";
import ExportMod from "../../model/exports/ExportMod";
import ThunderstoreMod from "../../model/ThunderstoreMod";
import ThunderstoreDownloaderProvider from "../../providers/ror2/downloading/ThunderstoreDownloaderProvider";
import InteractionProvider from "../../providers/ror2/system/InteractionProvider";
import { ProfileImportExport } from "../../r2mm/mods/ProfileImportExport";
import { valueToReadableDate } from "../../utils/DateUtils";
import * as ProfileUtils from "../../utils/ProfileUtils";
import { ModalCard } from "../all";
import ProfilesMixin from "../mixins/ProfilesMixin.vue";
import OnlineModList from "../views/OnlineModList.vue";


@Component({
    components: { OnlineModList, ModalCard}
})
export default class ImportProfileModal extends mixins(ProfilesMixin) {
    valueToReadableDate = valueToReadableDate;
    private importUpdateSelection: 'CREATE' | 'UPDATE' = 'CREATE';
    private importPhaseDescription: string = 'Downloading mods: 0%';
    private importViaCodeInProgress: boolean = false;
    private profileImportCode: string = '';
    private targetProfileName: string = '';
    private profileImportFilePath: string | null = null;
    private profileImportContent: ExportFormat | null = null;
    private activeStep:
        'FILE_CODE_SELECTION'
        | 'IMPORT_FILE'
        | 'IMPORT_CODE'
        | 'REVIEW_IMPORT'
        | 'IMPORT_UPDATE_SELECTION'
        | 'ADDING_PROFILE'
        | 'PROFILE_IS_BEING_IMPORTED'
    = 'FILE_CODE_SELECTION';

    get appName(): string {
        return ManagerInformation.APP_NAME;
    }

    get isOpen(): boolean {
        return this.$store.state.modals.isImportProfileModalOpen;
    }

    closeModal() {
        this.activeStep = 'FILE_CODE_SELECTION';
        this.targetProfileName = '';
        this.importUpdateSelection = 'CREATE';
        this.importViaCodeInProgress = false;
        this.profileImportCode = '';
        this.profileImportFilePath = null;
        this.profileImportContent = null;
        this.$store.commit('closeImportProfileModal');
    }

    get profileMods(): {known: ThunderstoreMod[], unknown: string[]} {
        const profileMods = this.profileImportContent ? this.profileImportContent.getMods() : [];
        const known: ThunderstoreMod[] = [];
        const unknown: string[] = [];

        for (const mod of profileMods) {
            const tsMod = this.$store.getters['tsMods/tsMod'](mod);
            if (tsMod) {
                known.push(tsMod);
            } else {
                unknown.push(mod.getName());
            }
        }

        unknown.sort();
        return {known, unknown};
    }

    // Fired when user selects to import either from file or code.
    async onFileOrCodeSelect(mode: 'FILE' | 'CODE') {
        if (mode === 'FILE') {
            this.activeStep = 'IMPORT_FILE';
            process.nextTick(async () => {
                const files = await InteractionProvider.instance.selectFile({
                    title: 'Import Profile',
                    filters: [{
                        name: "*",
                        extensions: ["r2z"]
                    }],
                    buttonLabel: 'Import'
                });
                await this.validateProfileFile(files);
            });
        } else {
            this.activeStep = 'IMPORT_CODE';
        }
    }

    // Fired when user has entered a profile code to import.
    async onProfileCodeEntered() {
        try {
            this.importViaCodeInProgress = true;
            const filepath = await ProfileImportExport.downloadProfileCode(this.profileImportCode.trim());
            await this.validateProfileFile([filepath]);
        } catch (e: any) {
            const err = R2Error.fromThrownValue(e, 'Failed to import profile');
            this.$store.commit('error/handleError', err);
        } finally {
            this.importViaCodeInProgress = false;
        }
    }

    // Check that selected profile zip is valid and proceed.
    async validateProfileFile(files: string[] | null) {
        if (files === null || files.length === 0) {
            this.closeModal();
            return;
        }

        let read: string = '';
        try {
            read = await ProfileUtils.readProfileFile(files[0]);
        } catch (e: unknown) {
            const err = R2Error.fromThrownValue(e);
            this.$store.commit('error/handleError', err);
            this.closeModal();
            return;
        }

        this.profileImportFilePath = files[0];
        try {
            this.profileImportContent = await ProfileUtils.parseYamlToExportFormat(read);
        } catch (e: unknown) {
            const err = R2Error.fromThrownValue(e);
            this.$store.commit('error/handleError', err)
            this.closeModal();
            return;
        }

        this.activeStep = 'REVIEW_IMPORT';
    }

    // Fired when user has accepted the mods to be imported in the review phase.
    async onProfileReviewConfirmed() {
        const profileContent = this.profileImportContent;
        const filePath = this.profileImportFilePath;

        // Check and return early for better UX.
        if (profileContent === null || filePath === null) {
            this.onContentOrPathNotSet();
            return;
        }

        this.targetProfileName = profileContent.getProfileName();
        this.activeStep = 'IMPORT_UPDATE_SELECTION';
    }

    // Fired when user selects whether to import a new profile or update existing one.
    onCreateOrUpdateSelect(mode: 'CREATE' | 'UPDATE') {
        this.importUpdateSelection = mode;

        if (mode === 'UPDATE') {
            this.targetProfileName = this.activeProfileName;
        }

        this.activeStep = 'ADDING_PROFILE';
    }

    // Fired when user confirms the import target location (new or existing profile).
    async onImportTargetSelected() {
        const profileContent = this.profileImportContent;
        const filePath = this.profileImportFilePath;

        // Recheck to ensure values haven't changed and to satisfy TypeScript.
        if (profileContent === null || filePath === null) {
            this.onContentOrPathNotSet();
            return;
        }

        const targetProfileName = this.makeProfileNameSafe(this.targetProfileName);

        // Sanity check, should not happen.
        if (targetProfileName === '') {
            const err = new R2Error("Can't import profile: unknown target profile", "Please try again");
            this.$store.commit('error/handleError', err);
            return;
        }

        if (this.importUpdateSelection === 'CREATE') {
            try {
                await this.$store.dispatch('profiles/addProfile', targetProfileName);
            } catch (e) {
                this.closeModal();
                const err = R2Error.fromThrownValue(e, 'Error while creating profile');
                this.$store.commit('error/handleError', err);
                return;
            }
        }

        await this.importProfile(targetProfileName, profileContent.getMods(), filePath);
    }

    async importProfile(targetProfileName: string, mods: ExportMod[], zipPath: string) {
        this.activeStep = 'PROFILE_IS_BEING_IMPORTED';
        this.importPhaseDescription = 'Downloading mods: 0%';
        const progressCallback = (progress: number|string) => typeof progress === "number"
            ? this.importPhaseDescription = `Downloading mods: ${Math.floor(progress)}%`
            : this.importPhaseDescription = progress;
        const game = this.$store.state.activeGame;
        const settings = this.$store.getters['settings'];
        const ignoreCache = settings.getContext().global.ignoreCache;
        const isUpdate = this.importUpdateSelection === 'UPDATE';

        try {
            const comboList = await ProfileUtils.exportModsToCombos(mods, game);
            await ThunderstoreDownloaderProvider.instance.downloadImportedMods(comboList, ignoreCache, progressCallback);
            await ProfileUtils.populateImportedProfile(comboList, mods, targetProfileName, isUpdate, zipPath, progressCallback);
        } catch (e) {
            await this.$store.dispatch('profiles/ensureProfileExists');
            this.closeModal();
            this.$store.commit('error/handleError', R2Error.fromThrownValue(e));
            return;
        }

        await this.$store.dispatch('profiles/setSelectedProfile', {profileName: targetProfileName, prewarmCache: true});
        this.closeModal();
    }

    onContentOrPathNotSet() {
        this.closeModal();
        const reason = `content is ${this.profileImportContent ? 'not' : ''} null, file path is ${this.profileImportFilePath ? 'not' : ''} null`;
        this.$store.commit('error/handleError', R2Error.fromThrownValue(`Can't install profile: ${reason}`));
    }
}
</script>

<template>
    <ModalCard v-if="activeStep === 'FILE_CODE_SELECTION'" key="FILE_CODE_SELECTION" :is-active="isOpen" @close-modal="closeModal">
        <template v-slot:header>
            <h2 class="modal-title" v-if="importUpdateSelection === 'CREATE'">How are you importing a profile?</h2>
            <h2 class="modal-title" v-if="importUpdateSelection === 'UPDATE'">How are you updating your profile?</h2>
        </template>
        <template v-slot:footer>
            <button id="modal-import-profile-file"
                    class="button is-info"
                    @click="onFileOrCodeSelect('FILE')">From file</button>
            <button id="modal-import-profile-code"
                    class="button is-primary"
                    @click="onFileOrCodeSelect('CODE')">From code</button>
        </template>
    </ModalCard>

    <ModalCard v-else-if="activeStep === 'IMPORT_FILE'" key="IMPORT_FILE" :is-active="isOpen" @close-modal="closeModal">
        <template v-slot:header>
            <h2 class="modal-title">Loading file</h2>
        </template>
        <template v-slot:footer>
            <p>A file selection window will appear. Once a profile has been selected it may take a few moments.</p>
        </template>
    </ModalCard>

    <ModalCard v-else-if="activeStep === 'IMPORT_CODE'" :can-close="!importViaCodeInProgress" key="IMPORT_CODE" :is-active="isOpen" @close-modal="closeModal">
        <template v-slot:header>
            <h2 class="modal-title">Enter the profile code</h2>
        </template>
        <template v-slot:body>
            <input
                v-model="profileImportCode"
                @keyup.enter="isProfileCodeValid(profileImportCode) && onProfileCodeEntered()"
                id="import-profile-modal-profile-code"
                class="input"
                type="text"
                ref="profileCodeInput"
                autocomplete="off"
            />
            <br />
            <br />
            <span class="tag is-dark" v-if="profileImportCode === ''">You haven't entered a code</span>
            <span class="tag is-danger" v-else-if="!isProfileCodeValid(profileImportCode)">Invalid code, check for typos</span>
            <span class="tag is-success" v-else>You may import the profile</span>
        </template>
        <template v-slot:footer>
            <button
                id="modal-import-profile-from-code-invalid"
                class="button is-danger"
                v-if="!isProfileCodeValid(profileImportCode)">
                Fix issues before importing
            </button>
            <button
                disabled
                id="modal-import-profile-from-code-loading"
                class="button is-disabled"
                v-else-if="importViaCodeInProgress">
                Loading...
            </button>
            <button
                id="modal-import-profile-from-code"
                class="button is-info"
                @click="onProfileCodeEntered();"
                v-else>
                Continue
            </button>
        </template>
    </ModalCard>

    <ModalCard v-else-if="activeStep === 'REVIEW_IMPORT'" key="REVIEW_IMPORT" :is-active="isOpen" @close-modal="closeModal">
        <template v-slot:header>
            <h2 class="modal-title">Packages to be installed</h2>
        </template>
        <template v-slot:body>
            <OnlineModList :paged-mod-list="profileMods.known" :read-only="true" />
            <div v-if="!profileMods.known.length || profileMods.unknown.length" class="notification is-warning margin-top">
                <p v-if="!profileMods.known.length">
                    None of the packages in the profile were found on Thunderstore:
                </p>
                <p v-else>
                    Some of the packages in the profile were not found on Thunderstore:
                </p>

                <p class="margin-top">{{ profileMods.unknown.join(", ") }}</p>

                <p v-if="profileMods.known.length" class="margin-top">
                    These packages will not be installed.
                </p>
                <p v-else class="margin-top">
                    Ensure the profile is intended for the currently selected game.
                </p>

                <p class="margin-top">
                    Updating the mod list from Thunderstore might solve this issue.

                    <span v-if="$store.state.tsMods.modsLastUpdated">
                        The mod list was last updated on {{ valueToReadableDate($store.state.tsMods.modsLastUpdated) }}.
                    </span>

                    <br />

                    <span v-if="$store.state.tsMods.isThunderstoreModListUpdateInProgress">
                        {{ $store.state.tsMods.thunderstoreModListUpdateStatus }}
                    </span>
                    <span v-else-if="$store.state.tsMods.thunderstoreModListUpdateError">
                        Error updating the mod list:
                        {{ $store.state.tsMods.thunderstoreModListUpdateError.message }}.
                        <a @click="$store.dispatch('tsMods/syncPackageList')">Retry</a>?
                    </span>
                    <span v-else>
                        Would you like to
                        <a @click="$store.dispatch('tsMods/syncPackageList')">update now</a>?
                    </span>
                </p>
            </div>
        </template>
        <template v-slot:footer>
            <div class="container is-flex is-justify-content-space-between">
                <button
                    id="modal-no-mods-to-import"
                    class="button is-danger"
                    @click="closeModal"
                >
                    Cancel import
                </button>
                <button
                    id="modal-review-confirmed"
                    class="button is-info"
                    :disabled="!profileMods.known.length"
                    @click="onProfileReviewConfirmed"
                >
                    Import
                </button>
            </div>
        </template>
    </ModalCard>

    <ModalCard v-else-if="activeStep === 'IMPORT_UPDATE_SELECTION'" key="IMPORT_UPDATE_SELECTION" :is-active="isOpen" @close-modal="closeModal">
        <template v-slot:header>
            <h2 class="modal-title">Are you going to be updating an existing profile or creating a new one?</h2>
        </template>
        <template v-slot:footer>
            <button id="modal-import-new-profile"
                    class="button is-info"
                    @click="onCreateOrUpdateSelect('CREATE')">
                Import new profile
            </button>
            <button id="modal-update-existing-profile"
                    class="button is-primary"
                    @click="onCreateOrUpdateSelect('UPDATE')">
                Update existing profile
            </button>
        </template>
    </ModalCard>

    <ModalCard v-else-if="activeStep === 'ADDING_PROFILE'" key="ADDING_PROFILE" :is-active="isOpen" @close-modal="closeModal">
        <template v-slot:header>
            <h2 v-if="importUpdateSelection === 'CREATE'" class="modal-title">Import a profile</h2>
        </template>
        <template v-slot:body v-if="importUpdateSelection === 'CREATE'">
            <p>This profile will store its own mods independently from other profiles.</p>
            <br/>
            <input
                v-model="targetProfileName"
                id="import-profile-modal-new-profile-name"
                class="input"
                type="text"
                ref="profileNameInput"
                autocomplete="off"
            />
            <br/><br/>
            <span class="tag is-dark" v-if="makeProfileNameSafe(targetProfileName) === ''">
                Profile name required
            </span>
            <span class="tag is-success" v-else-if="!doesProfileExist(targetProfileName)">
                "{{makeProfileNameSafe(targetProfileName)}}" is available
            </span>
            <span class="tag is-danger" v-else>
                "{{makeProfileNameSafe(targetProfileName)}}" is either already in use, or contains invalid characters
            </span>
        </template>
        <template v-slot:body v-else-if="importUpdateSelection === 'UPDATE'">
            <div class="notification is-warning">
                <p>All contents of the profile will be overwritten with the contents of the code/file.</p>
            </div>
            <p>Select a profile below:</p>
            <br/>
            <select class="select" v-model="targetProfileName">
                <option v-for="profile of profileList" :key="profile">{{ profile }}</option>
            </select>
        </template>
        <template v-slot:footer v-if="importUpdateSelection === 'CREATE'">
            <button id="modal-create-profile-invalid" class="button is-danger" v-if="doesProfileExist(targetProfileName)">Create</button>
            <button id="modal-create-profile" class="button is-info" v-else @click="onImportTargetSelected()">Create</button>
        </template>
        <template v-slot:footer v-else-if="importUpdateSelection === 'UPDATE'">
            <button id="modal-update-profile-invalid" class="button is-danger" v-if="!doesProfileExist(targetProfileName)">Update profile: {{ targetProfileName }}</button>
            <button id="modal-update-profile" class="button is-info" v-else @click="onImportTargetSelected()">Update profile: {{ targetProfileName }}</button>
        </template>
    </ModalCard>

    <ModalCard v-else-if="activeStep === 'PROFILE_IS_BEING_IMPORTED'" key="PROFILE_IS_BEING_IMPORTED" :is-active="isOpen" :canClose="false">
        <template v-slot:header>
            <h2 class="modal-title">{{importPhaseDescription}}</h2>
        </template>
        <template v-slot:footer>
            <p>
                This may take a while, as files are being downloaded, extracted, and copied.
                <br><br>
                Please do not close {{appName}}.
            </p>
        </template>
    </ModalCard>
</template>

<style scoped>
.modal-title {
    font-size: 1.5rem;
}
</style>
