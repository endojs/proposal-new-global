---
marp: true
theme: gaia
class: lead invert
paginate: true
---

## Problem statement

A way to evaluate a module and its dependencies in the context of a new global scope within the same Realm

<br>

> Tentatively updating the proposal name: proposal-module-global

<!-- visual customizations -->

<style>
/* justify, unless it's just one line (first===last) */
p {
  text-align: justify;
  text-align-last: center;
}
blockquote {
  border-left: 4px solid #888;
  padding-left: 1em;
  quotes: none;
  text-align: justify;
}
blockquote * {
  text-align: justify;
  text-align-last: left;
}
blockquote::before,
blockquote::after {
  content: none;
}
</style>
