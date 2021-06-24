<script>
	import { afterUpdate, createEventDispatcher, onMount } from "svelte";

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
	let containerContentLength //? Content length (Bounding Client Height / Width minus padding)
	let containerLength //? Actual length (Bounding Client Height / Width based on horizontal prop)

	const dispatch = createEventDispatcher()

	const dragging = { current: false } //? { current } to make sure values update throughout all the callbacks
	const internalPosition = { current: 0 } //? This is the position of the center of the scroll nub

	//? Replaces each attribute in an obj specified by the
	//? "keys" array with the value passed to the function
	//? This is not a pure function
	function replaceObjKeysWith(obj, keys, val) {
		for (const attr of keys) {
			obj[attr] = val
		}
	}

	//? Since I couldn't find a built-in for this I made one myself
	//? This goes through all the keys in the defaults object and checks
	//? if they exist in the target obj, if they don't or if their value is
	//? undefined, then it will create / replace that attribute's value with
	//? the one in the default object
	function setDefaultsByUndefined(obj, defaults) {
		const result = {...obj}

		for (const attr of Object.keys(defaults)) {
			if (result[attr] === undefined) {
				result[attr] = defaults[attr]
			}
		}

		return result
	}

	//? Styling Options

	//* Whether or not the Scrollbar should be horizontal
	export let horizontal = false

	//* General Styling
	export let styling = {}
	const defaultStyling = {
		width: '18px',
		height: '100%',
		padding: '4px',
		cssPosition: 'relative',
		top: 'unset',
		right: 'unset',
		left: 'unset',
		bottom: 'unset',
		hoverTransition: 'unset',
		borderRadius: '10px'
	}

	let renderedStyling
	$: {
		renderedStyling = setDefaultsByUndefined(styling, defaultStyling)
	}

	//* Color Scheme
	export let colorScheme = {}
	const defaultColors = {
		nubClicked: "#787878",
		nubHovered: "#A8A8A8",
		nub: "#C1C1C1",
		background: "#FFF",
	}

	let renderedColors
	$: {
		renderedColors = setDefaultsByUndefined(colorScheme, defaultColors)
	}

	//* Directly pass CSS rules for advanced users
	export let containerStyle = ""
	export let containerHoveredStyle = {}
	export let nubStyle = ""
	export let nubHoveredStyle = {}

	//? Props
	export let position
	let prevPosition

	$: {
		if (position !== prevPosition && containerLength) {
			internalPosition.current = (position / total) * containerLength
			prevPosition = position
		}
	}

	export let total
	export let viewable

	onMount(() => {
		containerBox = container.getBoundingClientRect()

		containerLength = (horizontal) ? containerBox.width : containerBox.height

		const computedContainerStyle = window.getComputedStyle(container)
		const paddingFront = (horizontal) ? computedContainerStyle.paddingLeft : computedContainerStyle.paddingTop
		const paddingBack = (horizontal) ? computedContainerStyle.paddingRight : computedContainerStyle.paddingBottom

		containerContentLength = containerLength -
							parseFloat(paddingFront) -
							parseFloat(paddingBack)

		const moveNub = (pos) => {
			//? Nub's dimensions
			const nubBox = nub.getBoundingClientRect()

			const nubLength = (horizontal) ? nubBox.width : nubBox.height
			const nubTop = (horizontal) ? nubBox.left : nubBox.top

			//? Center of the nub
			const center = Math.round(nubLength / 2) + nubTop

			//? How far to move to align the center of the nub to the cursor
			//? Because it makes most sense for the center of the nub to be
			//? aligned to your cursor instead of the top
			const delta = pos - center

			//? internalPosition represents how far the nub moved measured
			//? (from it's top to the top of the containertop)
			internalPosition.current += delta

			//? Limit internalPosition between 0 and the bottom of the container minus
			//? the nub's height because internalPosition represents the position
			//? at the top of the nub (otherwise the nub will only be stopped when
			//? its top touches the bottom)
			internalPosition.current = clamp(
				internalPosition.current, 0,
				containerContentLength - nubLength
			)

			dispatch('scroll', {
				//? Calculate the percentage scrolled but subtract the nub's height
				//? this is because we clamped the internalPosition, which is the position
				//? of the top of the nub to the total height minus the nub's height

				//? Without subtracting it here as well the percentage will end up
				//? being innacurate because it does not match up with the clamped range
				truePosition: internalPosition.current / (containerContentLength - nubLength),

				//? Here "false" position is still included because the someone using
				//? will probably want to keep a state of the scrollbar that is updated
				//? via event
				position: internalPosition.current / (containerContentLength),
			})
		}

		const onmousedown = (e) => {
			const boundingMax = (horizontal) ? containerBox.bottom : containerBox.right
			const boundingMin = (horizontal) ? containerBox.top : containerBox.left

			const mouseMeasure = (horizontal) ? e.y : e.x

			if (!(mouseMeasure < boundingMax && mouseMeasure > boundingMin)) {
				return
			}

			moveNub((horizontal) ? e.x : e.y)
			dragging.current = true
		}

		const onmouseup = () => {
			//? Reset Styles
			replaceObjKeysWith(container.style, Object.keys(containerHoveredStyle), "")
			replaceObjKeysWith(nub.style, Object.keys(nubHoveredStyle), "")
			nub.style.backgroundColor = ""

			dragging.current = false
		}

		const onmousemove = (e) => {
			if (dragging.current) {
				moveNub((horizontal) ? e.x : e.y)
			}
		}

		const ondrag = (e) => e.preventDefault()

		window.addEventListener('mousedown', onmousedown)
		window.addEventListener('mouseup', onmouseup)
		window.addEventListener('mousemove', onmousemove)

		//? To prevent dragging and totally breaking the scrollbar
		window.addEventListener('dragstart', ondrag)

		return () => {
			window.removeEventListener('mousedown', onmousedown)
			window.removeEventListener('mouseup', onmouseup)
			window.removeEventListener('mousemove', onmousemove)
			window.removeEventListener('dragstart', ondrag)
		}
	})

	afterUpdate(() => {
		if (dragging.current) {
			nub.style.backgroundColor = renderedColors.nubClicked

			Object.assign(container.style, containerHoveredStyle)
			Object.assign(nub.style, nubHoveredStyle)
		}
	})
</script>

<div
	class="container" bind:this={container}
	draggable="false"
	style="
		--nubColor: {renderedColors.nub};
		--nubHovered: {renderedColors.nubHovered};
		--backgroundColor: {renderedColors.background};

		/* Swap width and height if horizontal */
		--width: {(horizontal) ? renderedStyling.height : renderedStyling.width};
		--height: {(horizontal) ? renderedStyling.width : renderedStyling.height};
		--padding: {renderedStyling.padding};

		position: {renderedStyling.cssPosition};
		top: {renderedStyling.top};
		right: {renderedStyling.right};
		left: {renderedStyling.left};
		bottom: {renderedStyling.bottom};

		--hoverTransition: {renderedStyling.hoverTransition};
		--nubBorderRadius: {renderedStyling.borderRadius};
		{containerStyle}
	"
	on:mouseover={ () => Object.assign(container.style, containerHoveredStyle) }
	on:mouseout={ () => replaceObjKeysWith(container.style, Object.keys(containerHoveredStyle), "") }
>
	<div
		class="nub"
		style="
			{`${(horizontal) ? 'width' : 'height'}: ${viewable / total * 100}%;`}
			{`${(horizontal) ? 'left' : 'top'}: ${position / total * 100}%;`}
			{nubStyle}
		"
		draggable="false"
		on:mouseover={ () => Object.assign(nub.style, nubHoveredStyle) }
		on:mouseout={ () => replaceObjKeysWith(nub.style, Object.keys(nubHoveredStyle), "") }
		bind:this={nub}
	></div>
</div>

<style>
	.container {
		box-sizing: border-box;

		height: var(--height);
		width: var(--width);
		padding: var(--padding);

		background-color: var(--backgroundColor);
	}

	.nub {
		width: 100%;
		height: 100%;

		border-radius: var(--nubBorderRadius);

		position: relative;
		top: 0;

		background-color: var(--nubColor);
		transition: var(--hoverTransition);
	}

	.nub:hover {
		background-color: var(--nubHovered);
	}
</style>
