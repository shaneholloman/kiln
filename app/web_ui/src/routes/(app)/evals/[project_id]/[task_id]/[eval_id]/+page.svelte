<script lang="ts">
  import AppPage from "../../../../app_page.svelte"
  import type { Eval } from "$lib/types"
  import { client } from "$lib/api_client"
  import { KilnError, createKilnError } from "$lib/utils/error_handlers"
  import { onMount, tick } from "svelte"
  import { page } from "$app/stores"
  import type { EvalProgress } from "$lib/types"
  import InfoTooltip from "$lib/ui/info_tooltip.svelte"
  import { eval_config_to_ui_name } from "$lib/utils/formatters"
  import {
    model_info,
    load_model_info,
    model_name,
    prompt_name_from_id,
    current_task_prompts,
  } from "$lib/stores"
  import type { ProviderModels, PromptResponse } from "$lib/types"
  import { goto } from "$app/navigation"
  import { prompt_link } from "$lib/utils/link_builder"
  import { progress_ui_state } from "$lib/stores/progress_ui_store"
  import PropertyList from "$lib/ui/property_list.svelte"
  import EditDialog from "$lib/ui/edit_dialog.svelte"

  $: project_id = $page.params.project_id
  $: task_id = $page.params.task_id
  $: eval_id = $page.params.eval_id

  let evaluator: Eval | null = null
  let eval_error: KilnError | null = null
  let eval_loading = true

  let eval_progress_loading = true
  let eval_progress: EvalProgress | null = null
  let eval_progress_error: KilnError | null = null

  $: loading = eval_loading || eval_progress_loading
  $: error = eval_error || eval_progress_error

  onMount(async () => {
    // Wait for page params to load
    await tick()
    // can be async
    load_model_info()
    // Load data in parallel
    await Promise.all([get_eval(), get_eval_progress()])
  })

  async function get_eval() {
    try {
      eval_loading = true
      const { data, error } = await client.GET(
        "/api/projects/{project_id}/tasks/{task_id}/eval/{eval_id}",
        {
          params: {
            path: {
              project_id,
              task_id,
              eval_id,
            },
          },
        },
      )
      if (error) {
        throw error
      }
      evaluator = data
    } catch (error) {
      eval_error = createKilnError(error)
    } finally {
      eval_loading = false
    }
  }

  async function get_eval_progress() {
    eval_progress = null
    eval_progress_loading = true
    try {
      eval_progress = null
      const { data, error } = await client.GET(
        "/api/projects/{project_id}/tasks/{task_id}/eval/{eval_id}/progress",
        {
          params: {
            path: {
              project_id,
              task_id,
              eval_id,
            },
          },
        },
      )
      if (error) {
        throw error
      }
      eval_progress = data
    } catch (error) {
      eval_progress_error = createKilnError(error)
    } finally {
      eval_progress_loading = false
    }
  }

  type UiProperty = {
    name: string
    value: string
    tooltip?: string
    link?: string
  }

  function get_eval_properties(
    evaluator: Eval | null,
    eval_progress: EvalProgress | null,
    modelInfo: ProviderModels | null,
    taskPrompts: PromptResponse | null,
  ): UiProperty[] {
    if (!evaluator) {
      return []
    }
    const properties: UiProperty[] = []

    properties.push({
      name: "Name",
      value: evaluator.name,
    })
    if (evaluator.description) {
      properties.push({
        name: "Description",
        value: evaluator.description,
      })
    }
    if (evaluator.template) {
      properties.push({
        name: "Template",
        value: evaluator.template,
        tooltip: "The template used to create this eval.",
      })
    }
    properties.push({
      name: "ID",
      value: evaluator.id || "unknown",
    })

    let eval_set_size = ""
    if (eval_progress) {
      eval_set_size = " (" + eval_progress.dataset_size + " items)"
    }
    properties.push({
      name: "Eval Dataset",
      value: evaluator.eval_set_filter_id + eval_set_size,
      link: link_from_filter_id(evaluator.eval_set_filter_id),
    })
    let golden_dataset_size = ""
    if (eval_progress) {
      golden_dataset_size = " (" + eval_progress.golden_dataset_size + " items)"
    }
    properties.push({
      name: "Golden Dataset",
      value: evaluator.eval_configs_filter_id + golden_dataset_size,
      tooltip:
        "This is the dataset that we use to evaluate the quality of judge models. Items in this set need human ratings so we can compare judge ratings to human ratings.",
      link: link_from_filter_id(evaluator.eval_configs_filter_id),
    })

    if (eval_progress?.current_eval_method) {
      properties.push({
        name: "Judge Algorithm",
        value: eval_config_to_ui_name(
          eval_progress.current_eval_method.config_type,
        ),
        tooltip: "The evaluation algorithm used by your selected judge.",
      })
      properties.push({
        name: "Judge Model",
        value: model_name(
          eval_progress.current_eval_method.model_name,
          modelInfo,
        ),
        tooltip: "The model used by your selected judge.",
      })
    }

    if (eval_progress?.current_run_method) {
      properties.push({
        name: "Run Model",
        value: model_name(
          eval_progress.current_run_method.run_config_properties.model_name,
          modelInfo,
        ),
        tooltip: "The model used by your selected run method.",
      })
      properties.push({
        name: "Run Prompt",
        value:
          eval_progress.current_run_method.prompt?.name ||
          prompt_name_from_id(
            eval_progress.current_run_method.run_config_properties.prompt_id,
            taskPrompts,
          ),
        tooltip: "The prompt used by your selected run method.",
        link: prompt_link(
          project_id,
          task_id,
          eval_progress.current_run_method.run_config_properties.prompt_id,
        ),
      })
    }

    return properties
  }

  $: has_default_eval_config = evaluator && evaluator.current_config_id
  $: has_default_run_config = evaluator && evaluator.current_run_config_id

  let edit_dialog: EditDialog | null = null

  const MIN_DATASET_SIZE = 25
  let current_step: 0 | 1 | 2 | 3 | 4 | 5 | 6 = 0
  let required_more_eval_data = false
  let required_more_golden_data = false
  let goals: string[] = []
  let golden_dataset_explanation = ""
  const step_titles: string[] = [
    "Define Goals",
    "Create Eval Data",
    "Human Ratings",
    "Find the Best Judge",
    "Find the Best Way to Run this Task",
  ]
  const step_tooltips: Record<number, string> = {
    1: "Each eval needs a set of quality goals to measure (aka 'eval scores'). You can add separate evals for different goals, or multiple goals to the same eval.",
    2: "Each eval needs two datasets: one for ensuring the eval works (eval set), and another to help find the best way of running your task (golden set). We'll help you create both with synthetic data!",
    3: "A 'golden' dataset is a dataset of items that are rated by humans. Rating a 'golden' dataset lets us determine if the judge is working by checking how well it aligns to human preferences. ",
    4: "Benchmark various judge methods (model+prompt+algorithm). We'll compare judges to your golden dataset to find the judge which best matches your human preferences.",
    5: "This tool will help your compare a variety of options for running this task and find the best one. You can compare different models, prompts, or fine-tunes.",
  }
  function update_eval_progress(
    progress: EvalProgress | null,
    evaluator: Eval | null,
  ) {
    update_golden_dataset_explanation(progress)
    current_step = 1
    if (!progress || !evaluator) {
      return
    }

    // Goals are setup. Generate friendly names for them.
    goals = []
    for (const output of evaluator.output_scores) {
      goals.push(output.name + " (" + output.type + ")")
    }

    current_step = 2
    required_more_eval_data = progress.dataset_size < MIN_DATASET_SIZE
    required_more_golden_data = progress.golden_dataset_size < MIN_DATASET_SIZE
    if (required_more_eval_data || required_more_golden_data) {
      return
    }

    current_step = 3
    if (golden_dataset_explanation) {
      return
    }

    current_step = 4
    if (!has_default_eval_config) {
      return
    }

    current_step = 5
    if (!has_default_run_config) {
      return
    }

    // Everything is setup!
    current_step = 6
  }
  $: update_eval_progress(eval_progress, evaluator)

  function update_golden_dataset_explanation(progress: EvalProgress | null) {
    if (!progress) {
      return
    }
    if (progress.golden_dataset_size == 0) {
      golden_dataset_explanation =
        "Your golden dataset is empty. Add data to your golden dataset to get started."
      return
    }
    let golden_dataset_rating_issues: string[] = []
    if (progress.golden_dataset_not_rated_count > 0) {
      golden_dataset_rating_issues.push(
        `${progress.golden_dataset_not_rated_count} item${progress.golden_dataset_not_rated_count == 1 ? " is" : "s are"} unrated`,
      )
    }
    if (progress.golden_dataset_partially_rated_count > 0) {
      golden_dataset_rating_issues.push(
        `${progress.golden_dataset_partially_rated_count} item${progress.golden_dataset_partially_rated_count == 1 ? " is" : "s are"} partially unrated`,
      )
    }
    if (golden_dataset_rating_issues.length > 0) {
      // Some golden dataset items are not fully rated.
      golden_dataset_explanation = `In your golden dataset ${golden_dataset_rating_issues.join(" and ")}. Fully rate all items to to get the best results from your eval.`
    } else {
      golden_dataset_explanation = ""
    }
  }

  function tag_from_filter_id(filter_id: string): string | undefined {
    if (filter_id.startsWith("tag::")) {
      return filter_id.replace("tag::", "")
    }
    return undefined
  }

  function link_from_filter_id(filter_id: string): string | undefined {
    const tag = tag_from_filter_id(filter_id)
    if (tag) {
      return `/dataset/${project_id}/${task_id}?tags=${tag}`
    }
    return undefined
  }

  function add_eval_data() {
    if (!evaluator) {
      alert("Unable to add eval data. Please try again later.")
      return
    }
    const eval_tag = tag_from_filter_id(evaluator?.eval_set_filter_id)
    const golden_tag = tag_from_filter_id(evaluator?.eval_configs_filter_id)
    if (!eval_tag || !golden_tag) {
      alert(
        "No eval or golden dataset tag found. If you're using a custom filter, please setup the dataset manually.",
      )
      return
    }

    const params = new URLSearchParams()
    params.set("reason", "eval")
    if (evaluator.template) {
      params.set("template_id", evaluator.template)
    }
    params.set("eval_id", `${project_id}::${task_id}::${eval_id}`)
    params.set("splits", `${eval_tag}:0.8,${golden_tag}:0.2`)
    params.set("eval_link", window.location.pathname)
    const url = `/dataset/${project_id}/${task_id}/add_data?${params.toString()}`
    show_progress_ui("When you're done adding data, ", 2)
    goto(url)
  }

  function show_progress_ui(body: string, step: number) {
    progress_ui_state.set({
      title: "Creating Eval",
      body,
      link: $page.url.pathname,
      cta: "return to the eval",
      progress: null,
      step_count: 5,
      current_step: step,
    })
  }

  function show_golden_dataset() {
    if (!evaluator) {
      return
    }
    const url = link_from_filter_id(evaluator.eval_configs_filter_id)
    if (!url) {
      return
    }

    show_progress_ui("When you're done rating, ", 3)
    goto(url)
  }

  function compare_eval_methods() {
    let url = `/evals/${project_id}/${task_id}/${eval_id}/eval_configs`
    show_progress_ui("When you're done comparing judges, ", 4)
    goto(url)
  }

  function compare_run_methods() {
    let url = `/evals/${project_id}/${task_id}/${eval_id}/compare_run_methods`
    show_progress_ui("When you're done comparing run methods, ", 5)
    goto(url)
  }
</script>

<div class="max-w-[1400px]">
  <AppPage
    title="Eval: {evaluator?.name || ''}"
    subtitle="Follow these steps to find the best way to evaluate and run your task"
    sub_subtitle="Read the Docs"
    sub_subtitle_link="https://docs.getkiln.ai/docs/evaluations"
    action_buttons={[
      {
        label: "Edit",
        handler: () => {
          edit_dialog?.show()
        },
      },
    ]}
  >
    {#if loading}
      <div class="w-full min-h-[50vh] flex justify-center items-center">
        <div class="loading loading-spinner loading-lg"></div>
      </div>
    {:else if error}
      <div
        class="w-full min-h-[50vh] flex flex-col justify-center items-center gap-2"
      >
        <div class="font-medium">Error Loading Evaluator</div>
        <div class="text-error text-sm">
          {error.getMessage() || "An unknown error occurred"}
        </div>
      </div>
    {:else if evaluator}
      <div class="flex flex-col xl:flex-row gap-8 xl:gap-16 mb-8">
        <div class="grow">
          <ul class="steps steps-vertical ml-4 overflow-x-hidden">
            {#each [1, 2, 3, 4, 5] as step}
              <li
                class="step {current_step >= step ? 'step-primary' : ''}"
                data-content={current_step == step
                  ? "●"
                  : current_step > step
                    ? "✓"
                    : ""}
              >
                <div
                  class="text-left py-3 min-h-[100px] flex flex-col place-content-center pl-4"
                >
                  <div class="font-medium">
                    {step_titles[step - 1]}
                    {#if step_tooltips[step]}
                      <InfoTooltip
                        tooltip_text={step_tooltips[step]}
                        position={step < 4 ? "bottom" : "top"}
                        no_pad={true}
                      />
                    {/if}
                  </div>
                  <div class="text-sm text-gray-500">
                    {#if step == 1 && goals.length > 0}
                      This eval has {goals.length} goals: {goals.join(", ")}.
                    {:else if step == 2}
                      <div>
                        <div class="mb-1">
                          {#if eval_progress && !required_more_eval_data && !required_more_golden_data}
                            You have {eval_progress?.dataset_size} eval items and
                            {eval_progress?.golden_dataset_size} golden items.
                          {:else if eval_progress && eval_progress.dataset_size == 0 && eval_progress.golden_dataset_size == 0}
                            Create data for this eval.
                          {:else if eval_progress && (required_more_eval_data || required_more_golden_data)}
                            You require additional eval data. You only have
                            {#if required_more_eval_data && required_more_golden_data}
                              {eval_progress?.dataset_size} eval items and {eval_progress?.golden_dataset_size}
                              golden items. We suggest at least {MIN_DATASET_SIZE}
                              items in each set.
                            {:else if required_more_eval_data}
                              {eval_progress?.dataset_size} eval items. We suggest
                              at least {MIN_DATASET_SIZE} items.
                            {:else if required_more_golden_data}
                              {eval_progress?.golden_dataset_size} golden items.
                              We suggest at least {MIN_DATASET_SIZE} items.
                            {/if}
                          {/if}
                        </div>
                        <button
                          class="btn btn-sm {current_step == 2
                            ? 'btn-primary'
                            : ''}"
                          on:click={add_eval_data}
                        >
                          Add Eval Data
                        </button>
                      </div>
                    {:else if step == 3}
                      <div class="mb-1">
                        {#if golden_dataset_explanation}
                          {golden_dataset_explanation}
                        {:else}
                          All items in your golden dataset are fully rated.
                        {/if}
                      </div>
                      <div>
                        {#if link_from_filter_id(evaluator.eval_configs_filter_id)}
                          <button
                            class="btn btn-sm {current_step == 3
                              ? 'btn-primary'
                              : ''}"
                            on:click={show_golden_dataset}
                          >
                            {golden_dataset_explanation ? "Rate" : "View"} Golden
                            Dataset
                          </button>
                        {:else}
                          <!-- We always use "tag::" so this shouldn't happen unless it's created by code. -->
                          Your golden dataset is filtered by
                          <span class="font-mono bg-gray-100 p-1"
                            >{evaluator.eval_configs_filter_id}</span
                          >. Please rate these entries in the
                          <a
                            class="link"
                            href={`/dataset/${project_id}/${task_id}`}
                            >dataset tab</a
                          >.
                        {/if}
                      </div>
                    {:else if step == 4}
                      <div class="mb-1">
                        {#if eval_progress?.current_eval_method}
                          You selected the judge '{eval_config_to_ui_name(
                            eval_progress.current_eval_method.config_type,
                          )}' using the model '{model_name(
                            eval_progress.current_eval_method.model_name,
                            $model_info,
                          )}'.
                        {:else}
                          Compare automated evals to find one that aligns with
                          your human preferences.
                        {/if}
                      </div>
                      <div>
                        <button
                          class="btn btn-sm {current_step == 4
                            ? 'btn-primary'
                            : ''}"
                          on:click={compare_eval_methods}
                        >
                          Compare Judges
                        </button>
                      </div>
                    {:else if step == 5}
                      <div class="mb-1">
                        {#if eval_progress?.current_run_method}
                          You've selected the model '{model_name(
                            eval_progress.current_run_method
                              .run_config_properties.model_name,
                            $model_info,
                          )}' with the prompt '{eval_progress.current_run_method
                            .prompt?.name ||
                            prompt_name_from_id(
                              eval_progress.current_run_method
                                .run_config_properties.prompt_id,
                              $current_task_prompts,
                            )}'.
                        {:else}
                          Compare models, prompts and fine-tunes to find the
                          most effective.
                        {/if}
                      </div>
                      <div>
                        <button
                          class="btn btn-sm {current_step == 5
                            ? 'btn-primary'
                            : ''}"
                          on:click={compare_run_methods}
                        >
                          Compare Run Methods
                        </button>
                      </div>
                    {/if}
                  </div>
                </div>
              </li>
            {/each}
          </ul>
        </div>

        <div class="w-72 2xl:w-96 flex-none">
          <PropertyList
            properties={get_eval_properties(
              evaluator,
              eval_progress,
              $model_info,
              $current_task_prompts,
            )}
            title="Evaluator Properties"
          />
        </div>
      </div>
    {/if}
  </AppPage>
</div>

<EditDialog
  bind:this={edit_dialog}
  name="Eval"
  patch_url={`/api/projects/${project_id}/tasks/${task_id}/eval/${eval_id}`}
  delete_url={`/api/projects/${project_id}/tasks/${task_id}/eval/${eval_id}`}
  fields={[
    {
      label: "Eval Name",
      description: "A name to identify this eval.",
      api_name: "name",
      value: evaluator?.name || "",
      input_type: "input",
    },
    {
      label: "Description",
      description: "A description of the eval for you and your team.",
      api_name: "description",
      value: evaluator?.description || "",
      input_type: "textarea",
      optional: true,
    },
  ]}
/>
