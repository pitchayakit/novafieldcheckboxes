<template>
    <default-field :field="field" full-width-content>
        <template slot="field">
            <div
                :style="{columnCount: this.field.columns}"
                class="w-full max-col-2"
            >
                <div
                    v-for="(label, option) in field.options"
                    :key="option"
                    class="flex mb-2"
                >
                    <checkbox
                        :value="option"
                        :checked="isChecked(option)"
                        @input="toggleOption(option)"
                        :disabled="isDisabled(option)"
                        :class="isDisabled(option) ? 'text-gray' : ''"
                        class="mr-2"
                    />
                    <label
                        :for="field.name"
                        v-text="label"
                        @click="toggleOption(option)"
                        class="w-full leading-tight"
                    ></label>
                </div>
            </div>
            <p v-if="hasError" class="my-2 text-danger">
                {{ firstError }}
            </p>
        </template>
    </default-field>
</template>

<script>
import { FormField, HandlesValidationErrors } from 'laravel-nova'

export default {
    mixins: [FormField, HandlesValidationErrors],

    props: ['resourceName', 'resourceId', 'field'],

    data () {
        return {
            developmentOptions: [],
            dependsOnValue: null,
            watcherDebounce: null,
            watcherDebounceTimeout: 200,
        }
    },

    created () {
        this.registerDependencyWatchers(this.$root);
    },

    methods: {
        registerDependencyWatchers(root) {
            root.$children.forEach(component => {
                if (this.componentIsDependency(component)) {
                    if (component.selectedResourceId !== undefined) {
                        // BelongsTo field
                        component.$watch('selectedResourceId', this.dependencyWatcher, {immediate: true});
                        this.dependencyWatcher(component.selectedResourceId);
                    } else if (component.value !== undefined) {
                        // Text based fields
                        component.$watch('value', this.dependencyWatcher, {immediate: true});
                        this.dependencyWatcher(component.value);
                    }
                }
                this.registerDependencyWatchers(component);
            })
        },

        componentIsDependency(component) {
            if (component.field === undefined) {
                return false;
            }

            return component.field.attribute === 'development';
        },

        dependencyWatcher(value) {
            clearTimeout(this.watcherDebounce);
            this.watcherDebounce = setTimeout(() => {
                if (value === this.dependsOnValue) {
                    return;
                }

                this.dependsOnValue = value;
                this.getDevelopmentOptions(value)

                this.watcherDebounce = null;
            }, this.watcherDebounceTimeout);
        },

        isChecked(option) {
            return this.value ? this.value.includes(option) : false
        },

        toggleOption(option) {
            if (this.isChecked(option)) {
                this.$set(this, 'value', this.value.filter(item => item != option))

                return
            }

            this.value.push(option)
        },

        /*
         * Set the initial, internal value for the field.
         */
        setInitialValue() {
          this.value = this.field.value || []
        },

        /**
         * Fill the given FormData object with the field's internal value.
         */
        fill(formData) {
          formData.append(this.field.attribute, this.value || [])
        },

        /**
         * Update the field's internal value.
         */
        handleChange(value) {
          this.value = value
        },

        getDevelopmentOptions(developmentId) {
            Nova.request(`/api/developments/${developmentId}/facilities?type=Nearby`).then((data) => {
                //Remove pre-checked of checkboxes
                if(this.developmentOptions) {
                    let optionDifference = this.value.filter(value => !this.developmentOptions.includes(value));
                    this.value = optionDifference
                }

                //Update development options
                this.developmentOptions = data.data
                
                //Add pre-checked of checkboxes
                if(this.developmentOptions) {
                    this.developmentOptions = this.developmentOptions.map(String)
                    this.value = this.value.concat(this.developmentOptions)
                }   
          });
        },

        isDisabled(option){
            return this.developmentOptions ? this.developmentOptions.includes(option) : false
        },
    }
}
</script>

<style>
    .max-col-2 {
        -moz-column-count: 2;
        -webkit-column-count: 2;
        column-count: 2;
        white-space: nowrap;
    }

    .text-gray {
        color: gray;
    }
</style>
