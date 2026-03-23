---
theme: seriph
base: /slides/
title: Asynchronous Programming in C++
class: text-center
highlighter: shiki
shikiSetup: |
  import { defineShikiSetup } from '@slidev/types'
drawings:
  persist: false
transition: fade
mdc: true
colorSchema: auto
fonts:
  sans: 'Roboto Condensed'
  serif: 'Fira Sans'
  mono: 'Jetbrains Mono' 
defaults:
   layout: default
layout: cover
background: /img/bg-blue-1.jpg
---

# Asynchronous Programming in C++

## Structured Concurrency

Instructor: Krystian Piękoś Ph.D.

<div class="logo">

![logo](/img/logo.png)

</div>

---
layout: default
---

# Course Information

- Training duration: 
    - 9:00 AM - 4:00 PM
    - Lunch: ???
- Training materials:
    - https://cpp-26.infotraining.pl
    - GIT repository
- Lunch & Coffee Breaks
- Evaluation and feedback

---
layout: default
---

# Syllabus

<v-clicks>

* Parallelism and concurrency
* Corotines in C++20
* Asynchronous programming with Coroutines
* Structured Concurrency
* Senders/Receivers in C++26

 </v-clicks>

---
src: ./pages/structured-concurrency.md
---

---
src: ./pages/coroutines.md
---

---
src: ./pages/stdexec.md
---

