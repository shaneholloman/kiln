<script lang="ts">
  import { SynthDataGuidanceDataModel } from "./synth_data_guidance_datamodel"
  import Dialog from "$lib/ui/dialog.svelte"
  import InfoTooltip from "$lib/ui/info_tooltip.svelte"

  export let guidance_data: SynthDataGuidanceDataModel
  $: selected_template = guidance_data.selected_template
  // reactive
  const splits = guidance_data.splits

  let edit_splits_dialog: Dialog | null = null

  let editable_splits: Array<{ tag: string; percent: number }> = []

  const gen_data_labels: Record<"training" | "eval", string> = {
    training: "Fine-tuning",
    eval: "Evaluation",
  }

  function get_gen_data_label(gen_type: "training" | "eval" | null): string {
    if (!gen_type) {
      return "Unknown"
    }
    return gen_data_labels[gen_type]
  }

  function edit_splits() {
    // Convert object to array format for easier editing, scaling to percentages
    editable_splits = Object.entries($splits).map(([tag, percent]) => ({
      tag,
      percent: percent * 100, // Convert decimal to percentage
    }))
    edit_splits_dialog?.show()
  }

  function add_split() {
    editable_splits = [...editable_splits, { tag: "", percent: 0 }]
  }

  function remove_split(index: number) {
    editable_splits = editable_splits.filter((_, i) => i !== index)
  }

  function get_total_percentage(
    editable_splits: Array<{ tag: string; percent: number }>,
  ): number {
    return editable_splits.reduce((sum, split) => sum + split.percent, 0)
  }

  function is_valid_splits(
    editable_splits: Array<{ tag: string; percent: number }>,
  ): boolean {
    const total = get_total_percentage(editable_splits)
    const has_empty_tags = editable_splits.some(
      (split) => split.tag.trim() === "",
    )
    const has_negative_percent = editable_splits.some(
      (split) => split.percent < 0,
    )

    return (
      (editable_splits.length === 0 || Math.abs(total - 100) < 0.0001) &&
      !has_empty_tags &&
      !has_negative_percent
    )
  }

  function save_splits(): boolean {
    // Convert back to object format, scaling back to decimals
    const new_splits: Record<string, number> = {}
    editable_splits.forEach((split) => {
      new_splits[split.tag] = split.percent / 100 // Convert percentage to decimal
    })
    guidance_data.splits.set(new_splits)
    edit_splits_dialog?.close()
    return true
  }

  function cancel_edit(): boolean {
    edit_splits_dialog?.close()
    return true
  }
</script>

{#if guidance_data.gen_type}
  <div class="card flex-row border px-6 py-3 my-2 shadow-sm text-sm">
    <div class="flex flex-row items-center gap-3 flex-grow">
      <div>
        <!-- Uploaded to: SVG Repo, www.svgrepo.com, Generator: SVG Repo Mixer Tools -->
        <svg
          class="h-6 w-6 text-gray-500"
          viewBox="0 0 24 24"
          fill="none"
          xmlns="http://www.w3.org/2000/svg"
        >
          <path
            d="M5 22V14M5 14V4M5 14L7.47067 13.5059C9.1212 13.1758 10.8321 13.3328 12.3949 13.958C14.0885 14.6354 15.9524 14.7619 17.722 14.3195L17.9364 14.2659C18.5615 14.1096 19 13.548 19 12.9037V5.53669C19 4.75613 18.2665 4.18339 17.5092 4.3727C15.878 4.78051 14.1597 4.66389 12.5986 4.03943L12.3949 3.95797C10.8321 3.33284 9.1212 3.17576 7.47067 3.50587L5 4M5 4V2"
            stroke="currentColor"
            stroke-width="1.5"
            stroke-linecap="round"
          />
        </svg>
      </div>
      <div class="flex flex-col">
        <div class="text-xs text-gray-500 uppercase font-medium">Goal</div>
        <div class="whitespace-nowrap">
          {get_gen_data_label(guidance_data.gen_type)}
          <InfoTooltip
            tooltip_text="The goal of the data generation task. This impacts the type of data that will be generated."
            no_pad={true}
          />
        </div>
      </div>
    </div>
    {#if $selected_template}
      <div
        class="border-l border-gray-200 self-stretch mx-4 border-[0.5px]"
      ></div>
      <div class="flex flex-row items-center gap-3 flex-grow">
        <div>
          <!-- Uploaded to: SVG Repo, www.svgrepo.com, Generator: SVG Repo Mixer Tools -->
          <svg
            class="h-6 w-6 text-gray-500"
            viewBox="0 0 24 24"
            fill="none"
            xmlns="http://www.w3.org/2000/svg"
          >
            <path
              d="M3 10C3 6.22876 3 4.34315 4.17157 3.17157C5.34315 2 7.22876 2 11 2H13C16.7712 2 18.6569 2 19.8284 3.17157C21 4.34315 21 6.22876 21 10V14C21 17.7712 21 19.6569 19.8284 20.8284C18.6569 22 16.7712 22 13 22H11C7.22876 22 5.34315 22 4.17157 20.8284C3 19.6569 3 17.7712 3 14V10Z"
              stroke="currentColor"
              stroke-width="1.5"
            />
            <path
              d="M8 10H16"
              stroke="currentColor"
              stroke-width="1.5"
              stroke-linecap="round"
            />
            <path
              d="M8 14H13"
              stroke="currentColor"
              stroke-width="1.5"
              stroke-linecap="round"
            />
          </svg>
        </div>

        <div class="flex flex-col">
          <div class="text-xs text-gray-500 uppercase font-medium">
            Template
          </div>
          <div class="whitespace-nowrap">
            {$selected_template}
            <InfoTooltip
              tooltip_text="A prompt template used to generate data. You can edit the template when generating topics, inputs, or outputs."
              no_pad={true}
            />
          </div>
        </div>
      </div>
    {/if}
    <div
      class="border-l border-gray-200 self-stretch mx-4 border-[0.5px]"
    ></div>
    <div class="flex flex-row items-center gap-3 flex-grow">
      <div>
        <!-- Uploaded to: SVG Repo, www.svgrepo.com, Generator: SVG Repo Mixer Tools -->
        <svg
          class="h-6 w-6 text-gray-500"
          viewBox="0 0 24 24"
          fill="none"
          xmlns="http://www.w3.org/2000/svg"
        >
          <path
            d="M4.72848 16.1369C3.18295 14.5914 2.41018 13.8186 2.12264 12.816C1.83509 11.8134 2.08083 10.7485 2.57231 8.61875L2.85574 7.39057C3.26922 5.59881 3.47597 4.70292 4.08944 4.08944C4.70292 3.47597 5.59881 3.26922 7.39057 2.85574L8.61875 2.57231C10.7485 2.08083 11.8134 1.83509 12.816 2.12264C13.8186 2.41018 14.5914 3.18295 16.1369 4.72848L17.9665 6.55812C20.6555 9.24711 22 10.5916 22 12.2623C22 13.933 20.6555 15.2775 17.9665 17.9665C15.2775 20.6555 13.933 22 12.2623 22C10.5916 22 9.24711 20.6555 6.55812 17.9665L4.72848 16.1369Z"
            stroke="currentColor"
            stroke-width="1.5"
          />
          <circle
            cx="8.60724"
            cy="8.87891"
            r="2"
            transform="rotate(-45 8.60724 8.87891)"
            stroke="currentColor"
            stroke-width="1.5"
          />
        </svg>
      </div>
      <div class="flex flex-col">
        <div class="text-xs text-gray-500 uppercase font-medium">Tags</div>
        <div>
          <button class="hover:underline text-left" on:click={edit_splits}>
            {#if Object.keys($splits).length == 0}
              No tag assignments
            {:else}
              {@const split_descriptions = Object.entries($splits).map(
                ([split, percent]) => `${split} (${percent * 100}%)`,
              )}
              {split_descriptions.join(", ")}
            {/if}
          </button>
        </div>
      </div>
    </div>
  </div>
{/if}

<Dialog
  title="Edit Tag Assignments"
  bind:this={edit_splits_dialog}
  action_buttons={[
    {
      label: "Cancel",
      action: cancel_edit,
    },
    {
      label: "Save",
      action: save_splits,
      disabled: !is_valid_splits(editable_splits),
      isPrimary: true,
    },
  ]}
>
  <div class="font-light mb-4 text-sm">
    Tags will be randomly assigned to saved data in the following proportions:
  </div>

  {#if editable_splits.length === 0}
    <div class="font-medium my-16 text-center">
      No tags
      <div class="text-gray-500 font-normal text-sm">
        Data will be saved without tags
      </div>
      <button
        on:click={add_split}
        class="btn btn-sm btn-primary btn-outline mx-auto mt-4"
      >
        + Add Tag
      </button>
    </div>
  {:else}
    <div class="space-y-2 mt-12">
      {#each editable_splits as split, index}
        <div class="flex items-center gap-2">
          <input
            type="text"
            bind:value={split.tag}
            placeholder="Tag name"
            class="flex-1 px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
          />
          <input
            type="number"
            bind:value={split.percent}
            min="0"
            max="100"
            placeholder="0"
            class="w-20 px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
          />
          <span class="text-sm text-gray-500">%</span>
          <button
            on:click={() => remove_split(index)}
            class="px-2 py-1 hover:text-error hover:bg-red-50 rounded"
            title="Remove tag"
          >
            ×
          </button>
        </div>
      {/each}
    </div>

    <div class="flex flex-row items-center mt-3 mb-12">
      <div class="flex-1">
        <button on:click={add_split} class="btn btn-sm btn-primary btn-outline">
          + Add Tag
        </button>
      </div>
      <div class="text-gray-500 text-right">
        Total: {get_total_percentage(editable_splits).toFixed(1)}%
        {#if Math.abs(get_total_percentage(editable_splits) - 100) >= 0.000001}
          <div class="text-sm text-error ml-2">Must total 100%</div>
        {/if}
        {#if editable_splits.some((split) => split.tag === "")}
          <div class="text-sm text-error ml-2">
            Tags must be non-empty strings
          </div>
        {/if}
        {#if editable_splits.some((split) => typeof split.percent !== "number")}
          <div class="text-sm text-error ml-2">Percentages must be numbers</div>
        {/if}
      </div>
    </div>
  {/if}
</Dialog>
