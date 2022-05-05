<template>
    <default-field :field="field" full-width-content>
        <template slot="field">
            <div
                :style="{columnCount: this.field.columns}"
                class="w-full max-col-2"
            >
                <div
                    v-for="(option) in options"
                    :key="option.id"
                    class="flex mb-2"
                >
                    <checkbox
                        :value="option.id"
                        :checked="isChecked(option.id)"
                        @input="toggleOption(option.id)"
                        :disabled="isDisabled(option.id)"
                        :class="isDisabled(option.id) ? 'text-gray' : ''"
                        class="mr-2"
                    />
                    <label
                        :for="field.name"
                        v-text="option.value"
                        @click="toggleOption(option.id)"
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
        if(this.resourceName === 'properties' && (this.field.attribute === 'nearby_facility_ids' || this.field.attribute === 'security_facility_ids'))
            this.registerDependencyWatchers(this.$root);

        this.options = Object.keys(
            this.field.options
        ).map((key) => {
            return { id: key, value: this.field.options[key] };
        });

        this.options.sort(function(a,b) {
            var x = a.value.toLowerCase();
            var y = b.value.toLowerCase();
            return x < y ? -1 : x > y ? 1 : 0;
        });
    },

    methods: {
        registerDependencyWatchers(root) {
            
            root.$children.forEach(component => {
                if (this.componentIsDependency(component)) {
                    if (component.selectedResourceId !== undefined) {

                        // BelongsTo field
                        component.$watch('selectedResource', this.dependencyWatcher);
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

        dependencyWatcher(data) {
            clearTimeout(this.watcherDebounce);

            this.watcherDebounce = setTimeout(() => {
                if (data.value === this.dependsOnValue) {
                    return;
                }

                this.dependsOnValue = data.value;
                if(data.value !== undefined) 
                    this.getDevelopmentOptions(data.value)
                else
                    this.developmentOptions = []

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
            let type = this.field.attribute == "nearby_facility_ids" ? 'Nearby' : 'Security';

            Nova.request(`/api/developments/${developmentId}/facilities?type=${type}`).then((data) => {
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
                    //Remvoe duplicate
                    this.value = [...new Set([...this.value,...this.developmentOptions])]
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
