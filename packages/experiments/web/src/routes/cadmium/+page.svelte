<script>
	import MainCanvas from "./mainCanvas.svelte"
	import { browser } from "$app/environment"
	import { onMount } from "svelte"
	import { slide } from "svelte/transition"
	import { quintOut } from "svelte/easing"
	import {
		project_rust,
		project,
		realization_rust,
		realization,
		active_workbench_index,
		workbench,
		outlined_solids,
		step_being_edited,
		new_realization_needed,
		sketch_being_edited
	} from "./stores.js"
	import { default as init, Project } from "cadmium"
	import StepContextMenu from "./stepContextMenu.svelte"
	import SolidContextMenu from "./solidContextMenu.svelte"
	import ExtrudeForm from "./extrudeForm.svelte"
	import SketchForm from "./sketchForm.svelte"
	import PlaneForm from "./planeForm.svelte"
	import { base } from "$app/paths"

	let num_steps_applied = 1000
	// let realization = {}

	let current_step = {}
	let main_canvas

	let stepContextMenu
	let solidContextMenu

	const onPageClick = (e) => {
		// To make context menu disappear when
		// mouse is clicked outside context menu
		stepContextMenu.hide()
		solidContextMenu.hide()
	}

	if (browser) {
		onMount(() => {
			init().then(() => {
				let p = new Project("First Project")
				project_rust.set(p)
				active_workbench_index.set(0)

				new_realization_needed.set(true)
			})
		})
	}

	let username = "mattferraro.dev"

	$: if ($new_realization_needed) {
		console.log("getting new realization")
		//console.log('raw form:', $project_rust.json)
		project.set(JSON.parse($project_rust.json))
		workbench.set($project.workbenches[$active_workbench_index])

		const max_step = $step_being_edited >= 0 ? $step_being_edited + 1 : 1000
		realization_rust.set($project_rust.get_realization(0, max_step))

		const realization_as_json = JSON.parse($realization_rust.to_json())
		realization.set(realization_as_json)
		new_realization_needed.set(false)
	}

	const create_new_sketch = () => {
		let message_obj = {
			NewSketch: {
				workbench_id: $active_workbench_index,
				sketch_name: "",
				plane_name: ""
			}
		}
		let result = $project_rust.send_message(JSON.stringify(message_obj))
		let new_sketch_name = JSON.parse(result).success.sketch_name

		sketch_being_edited.set(new_sketch_name)
		new_realization_needed.set(true)
		step_being_edited.set($workbench.history.length)
	}

	const create_new_sketch_old_example = () => {
		let messages = [
			{
				NewPointOnSketch: {
					workbench_id: 0,
					sketch_name: "Sketch 1",
					point_id: 100,
					x: -0.6,
					y: 0.5
				}
			},
			{
				NewPointOnSketch: {
					workbench_id: 0,
					sketch_name: "Sketch 1",
					point_id: 101,
					x: -0.2,
					y: 0.4
				}
			},
			{
				NewPointOnSketch: {
					workbench_id: 0,
					sketch_name: "Sketch 1",
					point_id: 102,
					x: -0.2,
					y: 0.1
				}
			},
			{
				NewPointOnSketch: {
					workbench_id: 0,
					sketch_name: "Sketch 1",
					point_id: 103,
					x: -0.6,
					y: 0.1
				}
			},
			{
				NewLineOnSketch: {
					workbench_id: 0,
					sketch_name: "Sketch 1",
					line_id: 100,
					start_point_id: 100,
					end_point_id: 101
				}
			},
			{
				NewLineOnSketch: {
					workbench_id: 0,
					sketch_name: "Sketch 1",
					line_id: 101,
					start_point_id: 101,
					end_point_id: 102
				}
			},
			{
				NewLineOnSketch: {
					workbench_id: 0,
					sketch_name: "Sketch 1",
					line_id: 102,
					start_point_id: 102,
					end_point_id: 103
				}
			},
			{
				NewLineOnSketch: {
					workbench_id: 0,
					sketch_name: "Sketch 1",
					line_id: 103,
					start_point_id: 103,
					end_point_id: 100
				}
			}
		]
		let overall_success = true
		for (let message_obj of messages) {
			let result = $project_rust.send_message(JSON.stringify(message_obj))
			if (result === "success") {
				console.log("success of message: ", result)
			} else {
				console.log("failure of message: ", result)
				overall_success = false
				break
			}
		}
	}

	let solving = false

	const step_sketch = () => {
		// console.log('Step sketch')
		let message_obj = { StepSketch: { workbench_id: 0, sketch_name: "Sketch 1", steps: 1 } }
		let result = $project_rust.send_message(JSON.stringify(message_obj))
		let max_change = parseFloat(result)

		project.set(JSON.parse($project_rust.json))

		return max_change < 0.000001
	}

	function call(fn, every, times, fn2) {
		var repeater = function () {
			let stop_early = fn()
			if (!stop_early && --times) {
				window.setTimeout(repeater, every)
			} else {
				console.log("done!")
				fn2()
			}
		}
		repeater() // start loop
	}

	const solve_sketch = () => {
		solving = true
		call(step_sketch, 20, 100, () => {
			solving = false
		})
	}
	const create_new_extrusion = () => {
		console.log("okay!")
	}

	const highlightSolid = (solid_id) => {
		outlined_solids.set([...$outlined_solids, solid_id])
	}

	const unhighlightSolid = (solid_id) => {
		outlined_solids.set($outlined_solids.filter((id) => id !== solid_id))
	}

	let actions = [
		{
			alt: "new sketch",
			src: "/actions/sketch_min.svg",
			text: "New Sketch",
			handler: create_new_sketch
		},
		{ alt: "extrude", src: "/actions/extrude_min.svg", handler: create_new_extrusion },
		{ alt: "plane", src: "/actions/plane_min.svg" },
		{ alt: "step", src: "/actions/step_min.svg", text: "Step", handler: step_sketch },
		{ alt: "solve", src: "/actions/solve_min.svg", text: "Solve", handler: solve_sketch }
	]

	let icon_mapping = {
		Sketch: "/actions/sketch_min.svg",
		Plane: "/actions/plane_min.svg",
		Point: "/actions/point_min_icon.svg",
		Extrusion: "/actions/extrude_min.svg"
	}

	const history_item_onclick = (item) => {
		if (item?.data?.type === "Plane") {
			main_canvas.setCameraViewPlane2(item)
		}
	}

	const history_item_on_dblclick = (item, item_index) => {
		if ($step_being_edited === item_index) {
			step_being_edited.set(-1)
			sketch_being_edited.set(null)
			new_realization_needed.set(true)
		} else {
			step_being_edited.set(item_index)
			new_realization_needed.set(true)
		}
	}
</script>

<div id="container" class="bg-gray-50 grid grid-cols-editor grid-rows-editor h-[100vh]">
	<header class="col-span-2 bg-gray-100">
		<div class="flex items-center gap-4">
			<div class="shrink-0 select-none">
				<img class="object-cover h-10 w-10 ml-4" alt="logo" src="{base}/cadmium_logo_min.svg" />
			</div>
			<div class="select-none">CADmium</div>
			<div class="text-xl font-medium">{$project.name || ""}</div>

			<div class="flex-grow flex flex-row-reverse gap-4 mr-4">
				<div>
					<a href="https://github.com/mattferraro/cadmium"
						><img class="h-6 w-6" src="/github-mark.svg" alt="github logo" /></a
					>
				</div>
				<div>{username}</div>
			</div>
		</div>
	</header>
	<toolbar class="col-span-2 flex items-center gap-1">
		{#each actions as action}
			<button
				class="inline-flex items-center {action.text === 'Solve' && solving ? 'bg-gray-400' : ''} hover:bg-gray-200 p-1"
				on:click={action.handler}
			>
				<img class="h-8 w-8" src={action.src} alt={action.alt} />{action.text ? action.text : ""}
			</button>
		{/each}
	</toolbar>
	<aside class="bg-gray-100">
		<div class="flex flex-col select-none">
			<div class="font-bold text-sm px-2 py-2">History ({$workbench.history.length})</div>
			<div>
				{#each $workbench.history as item, item_index}
					<div
						class="flex items-center text-sm hover:bg-sky-200"
						on:click={() => {
							history_item_onclick(item)
						}}
						on:keypress={() => {
							history_item_onclick(item)
						}}
						on:dblclick={() => {
							history_item_on_dblclick(item, item_index)
						}}
						on:contextmenu|preventDefault={(e) => {
							stepContextMenu.rightClickContextMenu(e)
						}}
						role="button"
						tabindex="0"
					>
						<img class="h-8 w-8 px-1" src={icon_mapping[item.data.type]} alt={item.name} />
						{item["name"]}
					</div>
					{#if item_index === $step_being_edited}
						<div transition:slide={{ delay: 0, duration: 300, easing: quintOut, axis: "y" }}>
							{#if item.data.type === "Extrusion"}
								<ExtrudeForm {item} />
							{/if}
							{#if item.data.type === "Sketch"}
								<SketchForm {item} />
							{/if}
							{#if item.data.type === "Plane"}
								<PlaneForm {item} />
							{/if}
						</div>
					{/if}
				{/each}
			</div>
		</div>
		<div class="flex flex-col select-none">
			<div class="font-bold text-sm px-2 py-2">
				Solids ({$realization.solids ? Object.keys($realization.solids).length : 0})
			</div>
			<div>
				{#if $realization.solids}
					{#each Object.keys($realization.solids) as solid_id}
						<!-- svelte-ignore a11y-no-static-element-interactions -->
						<div
							class="flex items-center text-sm hover:bg-sky-200"
							on:contextmenu|preventDefault={(e) => {
								solidContextMenu.rightClickContextMenu(e, solid_id)
							}}
							on:pointerenter={() => {
								highlightSolid(solid_id)
							}}
							on:pointerleave={() => {
								unhighlightSolid(solid_id)
							}}
						>
							<img class="h-8 w-8 px-1" src="{base}/actions/part.svg" alt="solid" />
							{solid_id}
						</div>
					{/each}
				{/if}
			</div>
		</div>

		<StepContextMenu bind:this={stepContextMenu} />
		<SolidContextMenu bind:this={solidContextMenu} />
	</aside>
	<main class="h-[100%]">
		<MainCanvas bind:this={main_canvas} />
	</main>
	<footer class="col-span-2">Footer</footer>
</div>

<svelte:head>
	<!-- You can change icon sets according to your taste. Change `class` value in `menuItems` above to represent your icons. -->
	<!-- <link rel="stylesheet" href="/icon/css/mfglabs_iconset.css"> -->
	<link
		rel="stylesheet"
		href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.1.1/css/all.min.css"
		integrity="sha512-KfkfwYDsLkIlwQp6LFnl8zNdLGxu9YAA1QvwINks4PhcElQSvqcyVLLD9aMhXd13uQjoXtEKNosOWaZqXgel0g=="
		crossorigin="anonymous"
		referrerpolicy="no-referrer"
	/>
</svelte:head>

<svelte:window on:click={onPageClick} />
