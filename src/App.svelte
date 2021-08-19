<script lang="ts">
  import { afterUpdate, onMount } from "svelte";
  import { quill } from "svelte-quill";
  import lzstr from "lz-string";

  const options = {
    modules: {
      toolbar: [
        [{ header: [1, 2, 3, 4, 5, 6, false] }, { font: [] }],
        [{ color: [] }, { background: [] }],
        ["bold", "italic", "underline", "strike"],
        [{ list: "ordered" }, { list: "bullet" }],
        ["code-block", { script: "sub" }, { script: "super" }, "formula"],
        [{ indent: "-1" }, { indent: "+1" }, { align: [] }], // outdent/indent
        ["image", "link", "clean"],
      ],
    },
    placeholder: "Type something...",
    theme: "snow",
  };

  let _frontEditorRefs = [];
  let _backEditorRefs = [];
  let reloadEditors = false;
  let state;
  clear();

  let savedState = window.localStorage.getItem("savedState");
  if (savedState) {
    state = JSON.parse(lzstr.decompressFromUTF16(savedState));
  }

  $: {
    switch (state.styles.sizeClass) {
      case "threeby5":
        state.cardsPerSheet = 4;
        break;
      case "threeby4":
        state.cardsPerSheet = 6;
        break;
      case "two5by3":
        state.cardsPerSheet = 8;
        break;
      case "twoby35":
        state.cardsPerSheet = 10;
        break;
    }
  }
  $: classes = `${state.styles.sizeClass} ${state.styles.lineHeightClass} ${state.styles.cardSpacingClass} ${state.styles.borderColorClass} ${state.styles.borderSizeClass} ${state.styles.borderStyleClass} ${state.styles.borderSideClass} ${state.styles.cardPaddingClass}`;

  $: {
    window.localStorage.setItem(
      "savedState",
      lzstr.compressToUTF16(JSON.stringify(state))
    );
  }

  onMount(async () => {
    // state load text into quill editors
    for (let i = 0; i < state.cards.length; i++) {
      _frontEditorRefs[i].getElementsByClassName("ql-editor")[0].innerHTML =
        state.cards[i].front;
      _backEditorRefs[i].getElementsByClassName("ql-editor")[0].innerHTML =
        state.cards[i].back;
    }
  });

  afterUpdate(() => {
    if (reloadEditors) {
      reloadEditors = false;
      // state load text into quill editors
      for (let i = 0; i < state.cards.length; i++) {
        _frontEditorRefs[i].getElementsByClassName("ql-editor")[0].innerHTML =
          state.cards[i].front;
        _backEditorRefs[i].getElementsByClassName("ql-editor")[0].innerHTML =
          state.cards[i].back;
      }
    }
  });

  function addCard() {
    state.cards = [...state.cards, { front: "", back: "" }];
  }

  function print() {
    window.print();
  }

  function clear() {
    for (let i = 0; i < _frontEditorRefs.length; i++) {
      _frontEditorRefs[i].getElementsByClassName("ql-editor")[0].innerHTML = "";
      _backEditorRefs[i].getElementsByClassName("ql-editor")[0].innerHTML = "";
    }

    state = {
      cards: [{ front: "", back: "" }],
      cardsPerSheet: 6,
      styles: {
        sizeClass: "two5by3",
        cardSpacingClass: "cardspacing1",
        lineHeightClass: "lhnormal",
        borderColorClass: "bsblack",
        borderStyleClass: "bssolid",
        borderSizeClass: "bs1",
        borderSideClass: "bsboth",
        fontSizeClass: "fsnormal",
        cardPaddingClass: "cardpadding10",
      },
    };
  }

  function copyState() {
    navigator.permissions.query({ name: "clipboard-write" }).then((result) => {
      if (result.state == "granted" || result.state == "prompt") {
        const el = document.createElement("textarea");
        el.value = lzstr.compressToUTF16(JSON.stringify(state));
        document.body.appendChild(el);
        el.select();
        document.execCommand("copy");
        document.body.removeChild(el);
      }
    });
  }

  async function loadState() {
    const savedState = await navigator.clipboard.readText();
    state = JSON.parse(lzstr.decompressFromUTF16(savedState));

    reloadEditors = true;
  }

  function chunk(arr, len): any[][] {
    var chunks = [],
      i = 0,
      n = arr.length;

    while (i < n) {
      var chunk = new Array(len);
      for (let j = 0; j < len; j++) {
        chunk[j] = arr[j + i];
      }
      i += len;
      chunks.push(chunk);
    }

    return chunks;
  }

  function flipForPrint(arr): any[] {
    var output = new Array(arr.length);
    for (let i = 0; i < arr.length; i++) {
      if (i % 2 == 0) {
        output[i] = arr[i + 1];
      } else {
        output[i] = arr[i - 1];
      }
    }

    return output;
  }
</script>

<svelte:head>
  <link href="//cdn.quilljs.com/1.3.6/quill.snow.css" rel="stylesheet" />

  <link
    rel="stylesheet"
    href="https://cdn.jsdelivr.net/npm/katex@0.13.13/dist/katex.min.css"
    integrity="sha384-RZU/ijkSsFbcmivfdRBQDtwuwVqK7GMOw6IMvKyeWL2K5UAlyp6WonmB8m7Jd0Hn"
    crossorigin="anonymous"
  />
  <script
    defer
    src="https://cdn.jsdelivr.net/npm/katex@0.13.13/dist/katex.min.js"
    integrity="sha384-pK1WpvzWVBQiP0/GjnvRxV4mOb0oxFuyRxJlk6vVw146n3egcN5C925NCP7a7BY8"
    crossorigin="anonymous"></script>
</svelte:head>

<main class={state.styles.fontSizeClass}>
  <div class="editor-grid-container">
    <div class="header-cell-wide center">
      <button on:click={print}> Print </button>
      <button on:click={clear}> Clear </button>
      <button on:click={copyState}> Copy Save data to Clipboard </button>
      <button on:click={loadState}> Load Save data from Clipboard </button>
      <!-- <button on:click={() => alert(JSON.stringify(state))}> debug </button> -->
    </div>
    <div class="header-cell-wide">
      <h4>Card Size - you may need to print with narrow margins to fit</h4>
      <label>
        <input
          type="radio"
          bind:group={state.styles.sizeClass}
          name="cardSize"
          value="threeby5"
        />
        3" x 5" - 4 per page, print double sided landscape, flip on short edge (index
        card size)
      </label><br />
      <label>
        <input
          type="radio"
          bind:group={state.styles.sizeClass}
          name="cardSize"
          value="threeby4"
        />
        3" x 4" - 6 per page, print double sided portrait, flip on long edge
      </label><br />
      <label>
        <input
          type="radio"
          bind:group={state.styles.sizeClass}
          name="cardSize"
          value="two5by3"
        />
        2.5" x 3" - 8 per page, print double sided portrait, flip on long edge
      </label><br />
      <label>
        <input
          type="radio"
          bind:group={state.styles.sizeClass}
          name="cardSize"
          value="twoby35"
        />
        2" x 3.5" - 10 per page, print double sided portrait, flip on long edge (business
        card size)
      </label>
    </div>

    <div class="header-cell-wide">
      <h4>Border</h4>
      Style:
      <label>
        <input
          type="radio"
          bind:group={state.styles.borderStyleClass}
          name="borderStyle"
          value="bsnone"
        />
        None
      </label>
      <label>
        <input
          type="radio"
          bind:group={state.styles.borderStyleClass}
          name="borderStyle"
          value="bssolid"
        />
        Solid
      </label>
      <label>
        <input
          type="radio"
          bind:group={state.styles.borderStyleClass}
          name="borderStyle"
          value="bsdotted"
        />
        Dotted
      </label>
      <label>
        <input
          type="radio"
          bind:group={state.styles.borderStyleClass}
          name="borderStyle"
          value="bsdashed"
        />
        Dotted
      </label>
      <label>
        <input
          type="radio"
          bind:group={state.styles.borderStyleClass}
          name="borderStyle"
          value="bsdouble"
        />
        Double
      </label>
      <label>
        <input
          type="radio"
          bind:group={state.styles.borderStyleClass}
          name="borderStyle"
          value="bsgroove"
        />
        Groove
      </label>
      <label>
        <input
          type="radio"
          bind:group={state.styles.borderStyleClass}
          name="borderStyle"
          value="bsridge"
        />
        Ridge
      </label>
      <label>
        <input
          type="radio"
          bind:group={state.styles.borderStyleClass}
          name="borderStyle"
          value="bsinset"
        />
        Inset
      </label>
      <label>
        <input
          type="radio"
          bind:group={state.styles.borderStyleClass}
          name="borderStyle"
          value="bsoutset"
        />
        Outset
      </label>
      <label>
        <input
          type="radio"
          bind:group={state.styles.borderStyleClass}
          name="borderStyle"
          value="bscorners"
        />
        Corners Only (enable bg on print)
      </label>
      <br />
      Size:
      <label>
        <input
          type="radio"
          bind:group={state.styles.borderSizeClass}
          name="borderSize"
          value="bs1"
        />
        1px
      </label>
      <label>
        <input
          type="radio"
          bind:group={state.styles.borderSizeClass}
          name="borderSize"
          value="bs2"
        />
        2px
      </label>
      <label>
        <input
          type="radio"
          bind:group={state.styles.borderSizeClass}
          name="borderSize"
          value="bs3"
        />
        3px
      </label>
      <label>
        <input
          type="radio"
          bind:group={state.styles.borderSizeClass}
          name="borderSize"
          value="bs5"
        />
        5px
      </label>
      <br />
      Color:
      <label>
        <input
          type="radio"
          bind:group={state.styles.borderColorClass}
          name="borderColor"
          value="bsblack"
        />
        Black
      </label>
      <label>
        <input
          type="radio"
          bind:group={state.styles.borderColorClass}
          name="borderColor"
          value="bslightgrey"
        />
        Grey
      </label><br />
      Side:
      <label>
        <input
          type="radio"
          bind:group={state.styles.borderSideClass}
          name="borderSide"
          value="bsfront"
        />
        Front
      </label>
      <label>
        <input
          type="radio"
          bind:group={state.styles.borderSideClass}
          name="borderSide"
          value="bsback"
        />
        Back
      </label>
      <label>
        <input
          type="radio"
          bind:group={state.styles.borderSideClass}
          name="borderSide"
          value="bsboth"
        />
        Both
      </label>
    </div>

    <div class="header-cell">
      <h4>Line Height</h4>
      <label>
        <input
          type="radio"
          bind:group={state.styles.lineHeightClass}
          name="lineHeight"
          value="lhnarrow"
        />
        Narrow
      </label>
      <label>
        <input
          type="radio"
          bind:group={state.styles.lineHeightClass}
          name="lineHeight"
          value="lhnormal"
        />
        Normal
      </label>
      <label>
        <input
          type="radio"
          bind:group={state.styles.lineHeightClass}
          name="lineHeight"
          value="lhwide"
        />
        Wide
      </label>
      <label>
        <input
          type="radio"
          bind:group={state.styles.lineHeightClass}
          name="lineHeight"
          value="lhxwide"
        />
        XWide
      </label>
    </div>
    <div class="header-cell">
      <h4>Card Spacing</h4>
      <label>
        <input
          type="radio"
          bind:group={state.styles.cardSpacingClass}
          name="cardSpacing"
          value="cardspacing0"
        />
        0px
      </label>
      <label>
        <input
          type="radio"
          bind:group={state.styles.cardSpacingClass}
          name="cardSpacing"
          value="cardspacing1"
        />
        1px
      </label>
      <label>
        <input
          type="radio"
          bind:group={state.styles.cardSpacingClass}
          name="cardSpacing"
          value="cardspacing2"
        />
        2px
      </label>
      <label>
        <input
          type="radio"
          bind:group={state.styles.cardSpacingClass}
          name="cardSpacing"
          value="cardspacing3"
        />
        3px
      </label>
      <label>
        <input
          type="radio"
          bind:group={state.styles.cardSpacingClass}
          name="cardSpacing"
          value="cardspacing5"
        />
        5px
      </label>
    </div>
    <div class="header-cell">
      <h4>Font Sizes Override</h4>
      <label>
        <input
          type="radio"
          bind:group={state.styles.fontSizeClass}
          name="fontSizes"
          value="fssmall"
        />
        Small
      </label>
      <label>
        <input
          type="radio"
          bind:group={state.styles.fontSizeClass}
          name="fontSizes"
          value="fsnormal"
        />
        Normal
      </label>
      <label>
        <input
          type="radio"
          bind:group={state.styles.fontSizeClass}
          name="fontSizes"
          value="fslarge"
        />
        Large
      </label>
      <label>
        <input
          type="radio"
          bind:group={state.styles.fontSizeClass}
          name="fontSizes"
          value="fsxlarge"
        />
        XLarge
      </label>
      <label>
        <input
          type="radio"
          bind:group={state.styles.fontSizeClass}
          name="fontSizes"
          value="fsxxlarge"
        />
        XXLarge
      </label>
    </div>
    <div class="header-cell">
      <h4>Card Padding</h4>
      <label>
        <input
          type="radio"
          bind:group={state.styles.cardPaddingClass}
          name="cardPadding"
          value="cardpadding0"
        />
        0px
      </label>
      <label>
        <input
          type="radio"
          bind:group={state.styles.cardPaddingClass}
          name="cardPadding"
          value="cardpadding5"
        />
        5px
      </label>
      <label>
        <input
          type="radio"
          bind:group={state.styles.cardPaddingClass}
          name="cardPadding"
          value="cardpadding10"
        />
        10px
      </label>
      <label>
        <input
          type="radio"
          bind:group={state.styles.cardPaddingClass}
          name="cardPadding"
          value="cardpadding15"
        />
        15px
      </label>
      <label>
        <input
          type="radio"
          bind:group={state.styles.cardPaddingClass}
          name="cardPadding"
          value="cardpadding20"
        />
        20px
      </label>
      <label>
        <input
          type="radio"
          bind:group={state.styles.cardPaddingClass}
          name="cardPadding"
          value="cardpadding25"
        />
        25px
      </label>
    </div>
    <div class="header-cell-wide" />
    <div class="header-cell-alt {classes}"><h3>Front</h3></div>
    <div class="header-cell-alt {classes}"><h3>Back</h3></div>
    {#each state.cards as { front, back }, i}
      <div class="editor-cell {classes}">
        <div
          bind:this={_frontEditorRefs[i]}
          class="editor"
          use:quill={options}
          on:text-change={(e) => {
            front = e.detail.html;
          }}
        />
      </div>

      <div class="editor-cell {classes}">
        <div
          bind:this={_backEditorRefs[i]}
          class="editor"
          use:quill={options}
          on:text-change={(e) => (back = e.detail.html)}
        />
      </div>
    {/each}
    <div class="header-cell-wide center">
      <button on:click={addCard}> Add Card </button>
    </div>
    <div class="header-cell-wide center">
      <h3>Preview:</h3>
      <h3>(page breaks added during print preview)</h3>
    </div>
    <br />
  </div>
  <div class="printarea">
    {#each chunk(state.cards, state.cardsPerSheet) as chunks, i}
      {#if i > 0}<div class="pagebreak" />{/if}
      {#each chunks as chunk, j}
        <div class="editor-cell {classes}">
          <div class="ql-container ql-snow editor">
            <div class="ql-editor">
              {#if chunk}
                {@html chunk.front}
              {/if}
            </div>
          </div>
        </div>
      {/each}
      <div class="pagebreak" />
      {#each flipForPrint(chunks) as chunk, k}
        <div class="editor-cell {classes} back">
          <div class="ql-container ql-snow editor">
            <div class="ql-editor">
              {#if chunk}
                {@html chunk.back}
              {/if}
            </div>
          </div>
        </div>
      {/each}
    {/each}
  </div>
  <div class="editor-grid-container">
    <br />
    Questions? Issues? Reach out at
    <a href="https://github.com/Sammmmm/FlashCardPrinter"
      >https://github.com/Sammmmm/FlashCardPrinter</a
    >
  </div>
</main>

<style>
  @media print {
    .editor-grid-container {
      display: none;
    }

    .pagebreak {
      page-break-before: always;
    }
  }
</style>
