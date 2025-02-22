<template>
    <div class="flex w-full justify-end items-center mx-3">
        <button
            v-if="hasImportAction"
            data-testid="import-action-confirm"
            dusk="run-import-action-button"
            @click.prevent="openConfirmationModal"
            class="btn btn-default btn-primary"
            :title="__('Run Import Action')"
        >
            <svg width="14" height="14" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24"><path fill="var(--white)" d="M13 5.41V17a1 1 0 0 1-2 0V5.41l-3.3 3.3a1 1 0 0 1-1.4-1.42l5-5a1 1 0 0 1 1.4 0l5 5a1 1 0 1 1-1.4 1.42L13 5.4zM3 17a1 1 0 0 1 2 0v3h14v-3a1 1 0 0 1 2 0v3a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-3z"></path></svg>
            <span>{{ __('Import') }} {{ __(resourceInformation.label) }}</span>
        </button>

        <!-- Action Confirmation Modal -->
        <!-- <portal to="modals"> -->
        <transition name="fade">
            <component
                :is="selectedAction.component"
                :working="working"
                v-if="confirmActionModalOpened"
                :selected-resources="selectedResources"
                :resource-name="resourceName"
                :action="selectedAction"
                :errors="errors"
                @confirm="executeAction"
                @close="confirmActionModalOpened = false"
            />
        </transition>
        <!-- </portal> -->
    </div>
</template>

<script>
import _ from 'lodash'
import { Errors, InteractsWithResourceInformation } from 'laravel-nova'

export default {
    mixins: [InteractsWithResourceInformation],

    props: {
        selectedResources: {
            type: [Array, String],
            default: () => 'all',
        },
        resourceName: String,
        endpoint: {
            type: String,
            default: null,
        },
        queryString: {
            type: Object,
            default: () => ({
                currentSearch: '',
                encodedFilters: '',
                currentTrashed: '',
                viaResource: '',
                viaResourceId: '',
                viaRelationship: '',
            }),
        },
    },

    data: () => ({
        working: false,
        errors: new Errors(),
        selectedActionKey: '',
        confirmActionModalOpened: false,
        actions: [],
    }),

    watch: {
        /**
         * Watch the actions property for changes.
         */
        actions() {
            this.selectedActionKey = ''
            this.initializeActionFields()
        },

    },

    /**
     * Mount the component and retrieve its initial data.
     */
    async created() {
        this.getActions()
    },

    methods: {
        /**
         * Get the actions available for the current resource.
         */
        getActions() {
            this.actions = []
            this.pivotActions = null
            return Nova.request()
                .get(`/nova-api/${this.resourceName}/actions`, {
                    params: {
                        viaResource: this.viaResource,
                        viaResourceId: this.viaResourceId,
                        viaRelationship: this.viaRelationship,
                        relationshipType: this.relationshipType,
                        import: true
                    },
                })
                .then(response => {
                    this.actions = _.filter(response.data.actions, action => {
                        return !action.onlyOnDetail
                    })
                })
        },

        /**
         * Confirm with the user that they actually want to run the selected action.
         */
        openConfirmationModal() {
            this.selectedActionKey = 'import-' + this.resourceName
            this.confirmActionModalOpened = true
        },

        /**
         * Close the action confirmation modal.
         */
        closeConfirmationModal() {
            this.selectedActionKey = ''
            this.confirmActionModalOpened = false
        },

        /**
         * Initialize all of the action fields to empty strings.
         */
        initializeActionFields() {
            _(this.allActions).each(action => {
                _(action.fields).each(field => {
                    field.fill = () => ''
                })
            })
        },

        /**
         * Execute the selected action.
         */
        executeAction() {
            this.working = true

            Nova.request({
                method: 'post',
                url: this.endpoint || `/nova-api/${this.resourceName}/action`,
                params: this.actionRequestQueryString,
                data: this.actionFormData(),
            })
                .then(response => {
                    this.confirmActionModalOpened = false
                    this.handleActionResponse(response.data)
                    this.working = false
                })
                .catch(error => {
                    this.working = false

                    if (error.response.status == 422) {
                        this.errors = new Errors(error.response.data.errors)
                    }
                })
        },

        /**
         * Gather the action FormData for the given action.
         */
        actionFormData() {
            return _.tap(new FormData(), formData => {
                formData.append('resources', this.selectedResources)
                formData.append('import', true)

                _.each(this.selectedAction.fields, field => {
                    field.fill(formData)
                })
            })
        },

        /**
         * Handle the action response. Typically either a message, download or a redirect.
         */
        handleActionResponse(response) {
            if (response.message) {
                this.$emit('actionExecuted')
                this.$toasted.show(response.message, { type: 'success' })
            } else if (response.deleted) {
                this.$emit('actionExecuted')
            } else if (response.danger) {
                this.$emit('actionExecuted')
                this.$toasted.show(response.danger, { type: 'error' })
            } else if (response.download) {
                let link = document.createElement('a')
                link.href = response.download
                link.download = response.name
                document.body.appendChild(link)
                link.click()
                document.body.removeChild(link)
            } else if (response.redirect) {
                window.location = response.redirect
            } else if (response.openInNewTab) {
                window.open(response.openInNewTab, '_blank')
            } else {
                this.$emit('actionExecuted')
                this.$toasted.show(this.__('The action ran successfully!'), { type: 'success' })
            }
        },
    },

    computed: {
        hasImportAction() {
            // before
            for (let i=0; i<this.actions.length; i++) {
                if(this.actions[i].uriKey == 'import-' + this.resourceName) {
                    return true
                }
            }
            return false
        },

        selectedAction() {
            if (this.selectedActionKey) {
                return _.find(this.allActions, a => a.uriKey == this.selectedActionKey)
            }
        },

        /**
         * Get the query string for an action request.
         */
        actionRequestQueryString() {
            return {
                action: this.selectedActionKey,
                search: this.queryString.currentSearch,
                filters: this.queryString.encodedFilters,
                trashed: this.queryString.currentTrashed,
                viaResource: this.queryString.viaResource,
                viaResourceId: this.queryString.viaResourceId,
                viaRelationship: this.queryString.viaRelationship,
            }
        },


        /**
         * Get all of the available actions.
         */
        allActions() {
            return this.actions
        },

    },

}
</script>
