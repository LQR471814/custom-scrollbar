## Custom Scrollbar

A custom scrollbar Svelte component.

### Usage

This example allows you to control the scrollbar with arrows keys and record / update the position of the scrollbar

```html
<script lang="ts">
    import Scrollbar from "./Scrollbar.svelte";

    let scrolled = 0 //? Record & update scrollbar position

    //? Control scrollbar with arrow keys
    window.addEventListener('keydown', (e) => {
        switch (e.code) {
            case "ArrowDown":
                scrolled += 10
                break
            case "ArrowUp":
                scrolled -= 10
                break
        }
    })
</script>

<main>
    <Scrollbar
        position={scrolled}
        viewable={50}
        total={1000}
        on:scroll={
            (e) => {
                scrolled = e.detail.position * 1000
            }
        }
    />
</main>
```

You can also style the scrollbar.

```html
<Scrollbar
    styling={{
        width: "5px",
        padding: "1px"
    }}

    colorScheme={{
        nubHovered: "#000000"
    }}

    containerStyle="
        /* Pass CSS rules straight to
            the container or the nub */

        background-color: pink;"
/>
```
