<script lang="ts">
  import { run } from "svelte/legacy";

  interface Props {
    show: boolean;
    closeable?: boolean;
    children?: import("svelte").Snippet;
  }

  let { show = $bindable(), closeable = true, children }: Props = $props();

  // TODO https://caniuse.com/wf-dialog-closedby not supported yet

  // Relies on external styling
  let modalDialog: HTMLDialogElement | undefined = $state(undefined);

  run(() => {
    if (modalDialog) {
      if (show) {
        modalDialog.showModal();
      } else {
        modalDialog.close();
      }
    }
  });

  function onClick(e: MouseEvent) {
    // only dismiss the modal when clicking outside of the inner dialog content, on the dialog itself.
    if (e.target == modalDialog) {
      e.stopPropagation();
      if (closeable) {
        show = false;
      }
    }
  }

  function onKeyDown(e: KeyboardEvent) {
    if (e.key == "Escape" || e.key == "Enter") {
      e.stopPropagation();
      if (closeable) {
        show = false;
      }
    }
  }
</script>

<!-- svelte-ignore a11y_no_noninteractive_element_interactions -->
<dialog
  bind:this={modalDialog}
  onclick={onClick}
  onkeydown={onKeyDown}
  closedby="none"
>
  <div>{@render children?.()}</div>
</dialog>

<style>
  div {
    max-width: 80vw;
    max-height: 80vh;
  }

  dialog::backdrop {
    backdrop-filter: blur(2px);
  }
</style>
