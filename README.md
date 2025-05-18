# Safari Video UA Shadow

`<ClientRouter/>` [`swapBodyElement`](https://github.com/withastro/astro/blob/dde8fd96cd6a7821b95f47ede7e30c36dad244d9/packages/astro/src/transitions/swap-functions.ts#L73) currently replaces with one parsed in a “dead” context. Any `<video>` elements in that new tree never go through WebKit’s live-HTML parser path, so their native UA controls (and load / play functions) are never initialized

[WebKit #2955](https://github.com/WebKit/WebKit/pull/2955) fixes this for innerHTML

so on [/testing/](./src/pages/testing.astro) we look at a few ways to do it

and on [/](./src/pages/index.astro) and [/2/](./src/pages/2.astro) we reproduce the current "bug" in Client Router


## ship to astro 

haven't thought this through but can try it out like this

```js
// node_modules/astro/dist/transitions/swap-functions.js


// function swapBodyElement(newElement, oldElement) {
//   oldElement.replaceWith(newElement);
//   for (const el of oldElement.querySelectorAll(`[${PERSIST_ATTR}]`)) {
//     const id = el.getAttribute(PERSIST_ATTR);
//     const newEl = newElement.querySelector(`[${PERSIST_ATTR}="${id}"]`);
//     if (newEl) {
//       newEl.replaceWith(el);
//       if (newEl.localName === "astro-island" && shouldCopyProps(el) && !isSameProps(el, newEl)) {
//         el.setAttribute("ssr", "");
//         el.setAttribute("props", newEl.getAttribute("props"));
//       }
//     }
//   }
// }
 function swapBodyElement(newElement, oldElement) {

  const oldBody = oldElement;
  const newBody = newElement;
  const html = newBody.innerHTML;

  const range = document.createRange();
  range.selectNodeContents(oldBody);
  const frag = range.createContextualFragment(html);

  for (const el of oldBody.querySelectorAll(`[${PERSIST_ATTR}]`)) {
    const id = el.getAttribute(PERSIST_ATTR);
    const placeholder = frag.querySelector(`[${PERSIST_ATTR}="${id}"]`);
    if (placeholder) {
      placeholder.replaceWith(el);
      if (
        placeholder.localName === 'astro-island' &&
        shouldCopyProps(el) &&
        !isSameProps(el, placeholder)
      ) {
        el.setAttribute('ssr', '');
        el.setAttribute('props', placeholder.getAttribute('props'));
      }
    }
  }
  oldBody.replaceChildren(frag);
}
```