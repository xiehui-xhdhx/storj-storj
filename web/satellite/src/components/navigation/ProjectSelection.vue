// Copyright (C) 2021 Storj Labs, Inc.
// See LICENSE for copying information.

<template>
    <div ref="projectSelection" class="project-selection">
        <div
            role="button"
            tabindex="0"
            class="project-selection__selected"
            :class="{ active: isDropdownShown }"
            aria-roledescription="project-selection"
            @keyup.enter="toggleSelection"
            @click.stop.prevent="toggleSelection"
        >
            <div class="project-selection__selected__left">
                <ProjectIcon class="project-selection__selected__left__image" />
                <p class="project-selection__selected__left__name" :title="projectName">{{ projectName }}</p>
                <p class="project-selection__selected__left__placeholder">Projects</p>
            </div>
            <ArrowImage class="project-selection__selected__arrow" />
        </div>
        <div v-if="isDropdownShown" v-click-outside="closeDropdown" class="project-selection__dropdown" :style="style">
            <div v-if="isLoading" class="project-selection__dropdown__loader-container">
                <VLoader width="30px" height="30px" />
            </div>
            <div v-else class="project-selection__dropdown__items">
                <div tabindex="0" class="project-selection__dropdown__items__choice" @click.prevent.stop="closeDropdown">
                    <div class="project-selection__dropdown__items__choice__mark-container">
                        <CheckmarkIcon class="project-selection__dropdown__items__choice__mark-container__image" />
                    </div>
                    <p class="project-selection__dropdown__items__choice__selected">
                        {{ selectedProject.name }}
                    </p>
                </div>
                <div
                    v-for="project in projects"
                    :key="project.id"
                    class="project-selection__dropdown__items__choice"
                    @click.prevent.stop="onProjectSelected(project.id)"
                    @keyup.enter="onProjectSelected(project.id)"
                >
                    <p class="project-selection__dropdown__items__choice__unselected">{{ project.name }}</p>
                </div>
            </div>
            <div tabindex="0" class="project-selection__dropdown__link-container" @click.stop="onManagePassphraseClick" @keyup.enter="onManagePassphraseClick">
                <PassphraseIcon />
                <p class="project-selection__dropdown__link-container__label">Manage Passphrase</p>
            </div>
            <div tabindex="0" class="project-selection__dropdown__link-container" @click.stop="onProjectsLinkClick" @keyup.enter="onProjectsLinkClick">
                <ManageIcon />
                <p class="project-selection__dropdown__link-container__label">Manage Projects</p>
            </div>
            <div tabindex="0" class="project-selection__dropdown__link-container" @click.stop="onCreateLinkClick" @keyup.enter="onCreateLinkClick">
                <CreateProjectIcon />
                <p class="project-selection__dropdown__link-container__label">Create new</p>
            </div>
        </div>
    </div>
</template>

<script lang="ts">
import { Component, Vue } from 'vue-property-decorator';

import { AnalyticsHttpApi } from '@/api/analytics';
import { RouteConfig } from '@/router';
import { APP_STATE_ACTIONS, PM_ACTIONS } from '@/utils/constants/actionNames';
import { PROJECTS_ACTIONS } from '@/store/modules/projects';
import { AnalyticsErrorEventSource, AnalyticsEvent } from '@/utils/constants/analyticsEventNames';
import { LocalData } from '@/utils/localData';
import { PAYMENTS_ACTIONS } from '@/store/modules/payments';
import { ACCESS_GRANTS_ACTIONS } from '@/store/modules/accessGrants';
import { BUCKET_ACTIONS } from '@/store/modules/buckets';
import { Project } from '@/types/projects';
import { User } from '@/types/users';
import { OBJECTS_MUTATIONS } from '@/store/modules/objects';
import { APP_STATE_DROPDOWNS, MODALS } from '@/utils/constants/appStatePopUps';
import { APP_STATE_MUTATIONS } from '@/store/mutationConstants';

import VLoader from '@/components/common/VLoader.vue';

import ProjectIcon from '@/../static/images/navigation/project.svg';
import ArrowImage from '@/../static/images/navigation/arrowExpandRight.svg';
import CheckmarkIcon from '@/../static/images/navigation/checkmark.svg';
import PassphraseIcon from '@/../static/images/navigation/passphrase.svg';
import ManageIcon from '@/../static/images/navigation/manage.svg';
import CreateProjectIcon from '@/../static/images/navigation/createProject.svg';

// @vue/component
@Component({
    components: {
        ArrowImage,
        CheckmarkIcon,
        ProjectIcon,
        PassphraseIcon,
        ManageIcon,
        CreateProjectIcon,
        VLoader,
    },
})
export default class ProjectSelection extends Vue {
    private FIRST_PAGE = 1;
    private dropdownYPos = 0;
    private dropdownXPos = 0;
    private readonly analytics: AnalyticsHttpApi = new AnalyticsHttpApi();

    public isLoading = false;

    public $refs!: {
        projectSelection: HTMLDivElement,
    };

    /**
     * Fetches projects related information and than toggles selection popup.
     */
    public async toggleSelection(): Promise<void> {
        if (this.isOnboardingTour) return;

        const selectionContainer = this.$refs.projectSelection.getBoundingClientRect();

        const FIVE_PIXELS = 5;
        const TWENTY_PIXELS = 20;
        this.dropdownYPos = selectionContainer.top - FIVE_PIXELS;
        this.dropdownXPos = selectionContainer.right - TWENTY_PIXELS;

        this.toggleDropdown();

        if (this.isLoading || !this.isDropdownShown) return;

        this.isLoading = true;

        try {
            await this.$store.dispatch(PROJECTS_ACTIONS.FETCH);
            await this.$store.dispatch(PROJECTS_ACTIONS.GET_LIMITS, this.$store.getters.selectedProject.id);
        } catch (error) {
            await this.$notify.error(error.message, AnalyticsErrorEventSource.NAVIGATION_PROJECT_SELECTION);
        } finally {
            this.isLoading = false;
        }
    }

    /**
     * Toggles project dropdown visibility.
     */
    public toggleDropdown(): void {
        this.$store.dispatch(APP_STATE_ACTIONS.TOGGLE_ACTIVE_DROPDOWN, APP_STATE_DROPDOWNS.SELECT_PROJECT);
    }

    /**
     * Fetches all project related information.
     * @param projectID
     */
    public async onProjectSelected(projectID: string): Promise<void> {
        await this.analytics.eventTriggered(AnalyticsEvent.NAVIGATE_PROJECTS);
        await this.$store.dispatch(PROJECTS_ACTIONS.SELECT, projectID);
        LocalData.setSelectedProjectId(projectID);
        await this.$store.dispatch(PM_ACTIONS.SET_SEARCH_QUERY, '');
        this.closeDropdown();

        this.$store.commit(OBJECTS_MUTATIONS.CLEAR);
        this.$store.commit(APP_STATE_MUTATIONS.UPDATE_ACTIVE_MODAL, MODALS.enterPassphrase);

        if (this.isBucketsView) {
            await this.$router.push(RouteConfig.Buckets.path).catch(() => {return; });

            return;
        }

        if (this.$route.name === RouteConfig.ProjectDashboard.name) {
            const now = new Date();
            const past = new Date();
            past.setDate(past.getDate() - 30);

            try {
                await Promise.all([
                    this.$store.dispatch(PROJECTS_ACTIONS.FETCH_DAILY_DATA, { since: past, before: now }),
                    this.$store.dispatch(PAYMENTS_ACTIONS.GET_PROJECT_USAGE_AND_CHARGES_CURRENT_ROLLUP),
                    this.$store.dispatch(PROJECTS_ACTIONS.GET_LIMITS, this.$store.getters.selectedProject.id),
                    this.$store.dispatch(BUCKET_ACTIONS.FETCH, this.FIRST_PAGE),
                ]);
            } catch (error) {
                await this.$notify.error(error.message, AnalyticsErrorEventSource.NAVIGATION_PROJECT_SELECTION);
            }

            return;
        }

        if (this.$route.name === RouteConfig.AccessGrants.name) {
            try {
                await this.$store.dispatch(ACCESS_GRANTS_ACTIONS.FETCH, this.FIRST_PAGE);
            } catch (error) {
                await this.$notify.error(error.message, AnalyticsErrorEventSource.NAVIGATION_PROJECT_SELECTION);
            }

            return;
        }

        if (this.$route.name === RouteConfig.Users.name) {
            try {
                await this.$store.dispatch(PM_ACTIONS.FETCH, this.FIRST_PAGE);
            } catch (error) {
                await this.$notify.error(error.message, AnalyticsErrorEventSource.NAVIGATION_PROJECT_SELECTION);
            }
        }
    }

    /**
     * Returns top and left position of dropdown.
     */
    public get style(): Record<string, string> {
        return { top: `${this.dropdownYPos}px`, left: `${this.dropdownXPos}px` };
    }

    /**
     * Indicates if current route is onboarding tour.
     */
    public get isOnboardingTour(): boolean {
        return this.$route.path.includes(RouteConfig.OnboardingTour.path);
    }

    /**
     * Returns selected project's name.
     */
    public get projectName(): string {
        return this.$store.getters.selectedProject.name;
    }

    /**
     * Indicates if dropdown is shown.
     */
    public get isDropdownShown(): boolean {
        return this.$store.state.appStateModule.appState.activeDropdown === APP_STATE_DROPDOWNS.SELECT_PROJECT;
    }

    /**
     * Returns projects list from store.
     */
    public get projects(): Project[] {
        return this.$store.getters.projectsWithoutSelected;
    }

    /**
     * Returns selected project from store.
     */
    public get selectedProject(): Project {
        return this.$store.getters.selectedProject;
    }

    /**
     * Closes select project dropdown.
     */
    public closeDropdown(): void {
        this.$store.dispatch(APP_STATE_ACTIONS.CLOSE_POPUPS);
    }

    /**
     * Route to projects list page.
     */
    public onProjectsLinkClick(): void {
        if (this.$route.name !== RouteConfig.ProjectsList.name) {
            this.analytics.pageVisit(RouteConfig.ProjectsList.path);
            this.analytics.eventTriggered(AnalyticsEvent.MANAGE_PROJECTS_CLICKED);
            this.$router.push(RouteConfig.ProjectsList.path);
        }

        this.closeDropdown();
    }

    /**
     * Toggles manage passphrase modal shown.
     */
    public onManagePassphraseClick(): void {
        this.$store.commit(APP_STATE_MUTATIONS.UPDATE_ACTIVE_MODAL, MODALS.manageProjectPassphrase);

        this.closeDropdown();
    }

    /**
     * Route to create project page.
     */
    public onCreateLinkClick(): void {
        if (this.$route.name !== RouteConfig.CreateProject.name) {
            this.analytics.eventTriggered(AnalyticsEvent.CREATE_NEW_CLICKED);

            const user: User = this.$store.getters.user;
            const ownProjectsCount: number = this.$store.getters.projectsCount;

            if (!user.paidTier && user.projectLimit === ownProjectsCount) {
                this.$store.commit(APP_STATE_MUTATIONS.UPDATE_ACTIVE_MODAL, MODALS.createProjectPrompt);
            } else {
                this.analytics.pageVisit(RouteConfig.CreateProject.path);
                this.$store.commit(APP_STATE_MUTATIONS.UPDATE_ACTIVE_MODAL, MODALS.createProject);
            }
        }

        this.closeDropdown();
    }

    /**
     * Indicates if current route is objects view.
     */
    private get isBucketsView(): boolean {
        const currentRoute = this.$route.path;

        return currentRoute.includes(RouteConfig.Buckets.path);
    }
}
</script>

<style scoped lang="scss">
    .project-selection {
        font-family: 'font_regular', sans-serif;
        position: static;
        width: 100%;

        &__selected {
            box-sizing: border-box;
            padding: 22px 32px;
            outline: none;
            border: none;
            border-left: 4px solid #fff;
            width: 100%;
            display: flex;
            align-items: center;
            justify-content: space-between;
            cursor: pointer;
            position: static;

            &__left {
                display: flex;
                align-items: center;
                max-width: calc(100% - 16px);
                color: var(--c-grey-6);

                &__name {
                    max-width: calc(100% - 24px - 16px);
                    font-size: 14px;
                    line-height: 20px;
                    margin-left: 24px;
                    white-space: nowrap;
                    overflow: hidden;
                    text-overflow: ellipsis;
                }

                &__placeholder {
                    display: none;
                }
            }

            &:hover {
                background-color: var(--c-grey-1);
                border-color: var(--c-grey-1);

                p {
                    color: var(--c-blue-3);
                }

                :deep(path) {
                    fill: var(--c-blue-3);
                }
            }

            &:focus {
                outline: none;
                border-color: var(--c-grey-1);
                background-color: var(--c-grey-1);
                color: var(--c-blue-3);

                p {
                    color: var(--c-blue-3);
                }

                :deep(path) {
                    fill: var(--c-blue-3);
                }
            }
        }

        &__dropdown {
            position: absolute;
            min-width: 240px;
            max-width: 240px;
            background-color: #fff;
            border: 1px solid var(--c-grey-2);
            box-shadow: 0 2px 16px rgb(0 0 0 / 10%);
            border-radius: 8px;
            z-index: 1;

            &__loader-container {
                margin: 10px 0;
                display: flex;
                align-items: center;
                justify-content: center;
                border-radius: 8px 8px 0 0;
            }

            &__items {
                overflow-y: auto;
                max-height: 250px;
                background-color: #fff;
                border-radius: 6px;

                &__choice {
                    display: flex;
                    align-items: center;
                    padding: 8px 16px;
                    cursor: pointer;
                    height: 32px;
                    border-radius: 8px 8px 0 0;

                    &__selected,
                    &__unselected {
                        font-size: 14px;
                        line-height: 20px;
                        color: #1b2533;
                        white-space: nowrap;
                        overflow: hidden;
                        text-overflow: ellipsis;
                    }

                    &__selected {
                        font-family: 'font_bold', sans-serif;
                        margin-left: 24px;
                    }

                    &__unselected {
                        padding-left: 40px;
                    }

                    &:hover {
                        background-color: #f5f6fa;

                        p {
                            color: var(--c-blue-3);
                        }
                    }

                    &__mark-container {
                        width: 16px;
                        height: 16px;

                        &__image {
                            object-fit: cover;
                        }
                    }

                    &:focus {
                        background-color: #f5f6fa;
                    }
                }
            }

            &__link-container {
                padding: 8px 16px;
                height: 32px;
                cursor: pointer;
                display: flex;
                align-items: center;
                border-top: 1px solid var(--c-grey-2);

                &__label {
                    font-size: 14px;
                    line-height: 20px;
                    color: var(--c-grey-6);
                    margin-left: 24px;
                }

                &:last-of-type {
                    border-radius: 0 0 8px 8px;
                }

                &:hover {
                    background-color: #f5f6fa;

                    p {
                        color: var(--c-blue-3);
                    }

                    :deep(path) {
                        fill: var(--c-blue-3);
                    }
                }

                &:focus {
                    background-color: #f5f6fa;
                }
            }
        }
    }

    .active {
        border-color: #000;

        p {
            color: var(--c-blue-6);
            font-family: 'font_bold', sans-serif;
        }

        :deep(path) {
            fill: #000;
        }
    }

    .active:hover {
        border-color: var(--c-blue-3);
        background-color: #f7f8fb;

        p {
            color: var(--c-blue-3);
        }

        :deep(path) {
            fill: var(--c-blue-3);
        }
    }

    @media screen and (max-width: 1280px) and (min-width: 500px) {

        .project-selection__selected {
            padding: 10px 0;
            justify-content: center;

            &__left {
                min-width: 18px;
                flex-direction: column;
                align-items: center;

                &__name {
                    display: none;
                }

                &__placeholder {
                    display: block;
                    margin: 10px 0 0;
                    font-family: 'font_medium', sans-serif;
                    font-size: 9px;
                }
            }

            &__arrow {
                display: none;
            }
        }

        .active p {
            font-family: 'font_medium', sans-serif;
        }
    }
</style>
