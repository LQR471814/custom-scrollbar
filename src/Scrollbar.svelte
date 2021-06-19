<svelte:options tag="js-table-scrollbar" />

<script>
	import { createEventDispatcher, onMount } from "svelte";

	function clamp(n, min, max) {
		if (n < min) {
			return min
		} else if (n > max) {
			return max
		} else {
			return n
		}
	}

	//? Variables
	let container
	let nub

	let containerBox
	let containerHeight //? Content height (Bounding Client Height minus padding)

	const dispatch = createEventDispatcher()

	const dragging = { current: false } //? { current } to make sure values update throughout all the callbacks
	const yPosition = { current: 0 } //? This is the position of the center of the scroll nub

	//? Styling Options
	export let styling = {
		width: '12px',
		padding: '4px',
		cssPosition: 'fixed',
		hoverTransition: '0.1s ease-in-out background-color',
	}

	export let colorScheme = {
		nubHovered: "#A8A8A8",
		nub: "#C1C1C1",
		background: "#FFF",
	}

	//? Directly pass CSS rules for advanced users
	export let containerStyle = ""
	export let nubStyle = ""

	//? Props
	export let position
	let prevPosition

	$: {
		if (position !== prevPosition && containerBox) {
			yPosition.current = (position / total) * containerBox.height
			prevPosition = position
		}
	}

	export let total
	export let viewable

	onMount(() => {
		containerBox = container.getBoundingClientRect()

		containerHeight = containerBox.height -
							parseFloat(window.getComputedStyle(container).paddingTop) -
							parseFloat(window.getComputedStyle(container).paddingBottom)

		const moveNub = (y) => {
			//? Nub's dimensions
			const nubBox = nub.getBoundingClientRect()

			//? Center of the nub
			const centerY = Math.round(nubBox.height / 2) + nubBox.top

			//? How far to move to align the center of the nub to the cursor
			//? Because it makes most sense for the center of the nub to be
			//? aligned to your cursor instead of the top
			const deltaY = y - centerY

			//? yPosition represents how far the nub moved measured
			//? (from it's top to the top of the containertop)
			yPosition.current += deltaY

			//? Limit yPosition between 0 and the bottom of the container minus
			//? the nub's height because yPosition represents the position
			//? at the top of the nub (otherwise the nub will only be stopped when
			//? its top touches the bottom)
			yPosition.current = clamp(
				yPosition.current, 0,
				containerHeight - nubBox.height
			)

			dispatch('scroll', {
				//? Calculate the percentage scrolled but subtract the nub's height
				//? this is because we clamped the yPosition, which is the position
				//? of the top of the nub to the total height minus the nub's height

				//? Without subtracting it here as well the percentage will end up
				//? being innacurate because it does not match up with the clamped range
				truePosition: yPosition.current / (containerHeight - nubBox.height),

				//? Here "false" position is still included because the someone using
				//? will probably want to keep a state of the scrollbar that is updated
				//? via event
				position: yPosition.current / (containerHeight),
			})

			//? Just regularly convert yPosition into a percentage
			// nub.style.top = `${yPosition.current / containerHeight * 100}%`
		}

		const onmousedown = (e) => {
			if (!(e.x < containerBox.right && e.x > containerBox.left)) {
				return
			}

			moveNub(e.y)
			dragging.current = true
		}

		const onmouseup = () => {
			dragging.current = false
		}

		const onmousemove = (e) => {
			if (dragging.current) {
				moveNub(e.y)
			}
		}

		window.addEventListener('mousedown', onmousedown)
		window.addEventListener('mouseup', onmouseup)
		window.addEventListener('mousemove', onmousemove)

		//? To prevent dragging and totally breaking the scrollbar
		window.addEventListener('dragstart', (e) => e.preventDefault())
	})
</script>

<div
	class="container" bind:this={container}
	draggable="false"
	style="
		--nubColor: {colorScheme.nub};
		--nubHovered: {colorScheme.nubHovered};
		--backgroundColor: {colorScheme.background};

		--width: {styling.width};
		--padding: {styling.padding};
		--position: {styling.cssPosition};
		--hoverTransition: {styling.hoverTransition};

		--nubHeight: {viewable / total * 100}%;
		{containerStyle}
	"
>
	<div
		class="nub"
		style="top: {position / total * 100}%; {nubStyle}"
		draggable="false"
		bind:this={nub}
	></div>
</div>

<style>
	.container {
		position: var(--position);

		top: 0;
		right: 0;

		height: calc(100vh - var(--padding) * 2);

		width: var(--width);
		padding: var(--padding);

		background-color: var(--backgroundColor);
	}

	.nub {
		width: 100%;
		border-radius: 10px;
		height: var(--nubHeight);

		position: relative;
		top: 0;

		background-color: var(--nubColor);
		transition: var(--hoverTransition);
	}

	.nub:hover {
		background-color: var(--nubHovered);
	}
</style>
