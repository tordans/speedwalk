<script lang="ts">
  interface Props {
    list: any[];
    idx?: number;
  }

  let { list, idx = $bindable(0) }: Props = $props();

  function prev() {
    if (idx != 0) {
      idx--;
    }
  }

  function next() {
    if (idx != list.length - 1) {
      idx++;
    }
  }
</script>

<div
  style="display: flex; justify-content: space-between; align-items: center;"
>
  <button disabled={idx == 0} onclick={prev}>Previous</button>
  {idx + 1} / {list.length}
  <button disabled={idx == list.length - 1} onclick={next}>Next</button>
</div>
