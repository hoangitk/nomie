<script>
  import { createEventDispatcher } from "svelte";
  import { navigate } from "svelte-routing";

  export let to = undefined;

  const dispatch = createEventDispatcher();
  const onClick = event => {
    if (to) {
      navigate(to);
    }
    dispatch("click", event);
  };
</script>

<button class="btn btn-clear text-faded" on:click={onClick}>
  <i class="zmdi zmdi-chevron-left mr-2 " />
  Back
</button>
