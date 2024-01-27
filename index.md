---
layout: default
title: Home
---

<h1>Billy Gigurtsis is <span id="typewriter"></span></h1>

Want to know more? Too bad, bitch.

<style>
/* Typewriter cursor effect */
#typewriter {
  border-right: .1em solid;
  white-space: nowrap;
  overflow: hidden;
}

/* Cursor animations */
@keyframes blink-caret {
  50% { border-color: transparent; }
}

.typewriter-cursor {
  animation: blink-caret 1s steps(1) infinite;
}
</style>

<script>
const phrases = [
  "a choreographer",
  "a security expert",
  "a performer",
  "a creative director",
  "an engineer"
];

const typewriterElement = document.getElementById('typewriter');
let phraseIndex = 0;
let letterIndex = 0;
let currentPhrase = [];
let isDeleting = false;
let isEnd = false;

function type() {
  if (isEnd) {
    return;
  }

  if (isDeleting) {
    currentPhrase.pop();
    letterIndex--;
  } else {
    currentPhrase.push(phrases[phraseIndex][letterIndex]);
    letterIndex++;
  }

  typewriterElement.textContent = currentPhrase.join('');

  if (!isDeleting && letterIndex === phrases[phraseIndex].length) {
    isDeleting = true;
    setTimeout(type, 2000); // Pause at end
  } else if (isDeleting && letterIndex === 0) {
    isDeleting = false;
    phraseIndex = (phraseIndex + 1) % phrases.length; // Move to the next phrase
    setTimeout(type, 200);
  } else {
    setTimeout(type, isDeleting ? 100 : 200); // Typing speed
  }
}

// Start typing effect after a delay
setTimeout(() => {
  typewriterElement.className = 'typewriter-cursor'; // Start cursor effect
  type();
}, 1000);
</script>
