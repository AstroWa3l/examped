<script>
  import { onMount, onDestroy } from "svelte";

  /** @type {{ question: string, options: string[], answer: string }[]} */
  let questions = [];
  let currentQuestionIndex = 0;
  /** @type {string | null} */
  let selectedAnswer = null;
  let isLoading = true;
  let score = 0;
  let hasSubmitted = false;
  let examFinished = false;
  let timeLeft = 3600; // one hour in seconds
  /** @type {number | undefined} */
  let timerId;

  // Reactive declaration to format timeLeft in mm:ss
  $: formattedTime = `${Math.floor(timeLeft / 60)}:${(timeLeft % 60)
    .toString()
    .padStart(2, "0")}`;

  async function fetchQuestions() {
    const res = await fetch("/questions.json");
    questions = await res.json();
    isLoading = false;
    // Set timer for the entire exam once questions are loaded
    timeLeft = 3600;
    startTimer();
  }

  function submitAnswer(auto = false) {
    // Avoid duplicate submission
    if (hasSubmitted) return;

    hasSubmitted = true;
    if (selectedAnswer === null) {
      console.log("No answer selected (time out).");
    } else {
      console.log("User selected:", selectedAnswer);
      if (selectedAnswer === questions[currentQuestionIndex].answer) {
        score += 1;
      }
    }
    // Automatically go to next question after a short delay
    setTimeout(() => {
      nextQuestion();
    }, 500);
  }

  function nextQuestion() {
    if (currentQuestionIndex < questions.length - 1) {
      currentQuestionIndex += 1;
      // Reset state for the next question without touching the timer
      selectedAnswer = null;
      hasSubmitted = false;
    } else {
      console.log("Quiz finished. Final score:", score);
      examFinished = true;
      clearInterval(timerId);
    }
  }

  function startTimer() {
    timerId = setInterval(() => {
      timeLeft -= 1;
      if (timeLeft <= 0) {
        clearInterval(timerId);
        examFinished = true;
        console.log("Time is up. Exam ended. Final score:", score);
      }
    }, 1000);
  }

  // Toggle dark mode by toggling a class on the body
  function toggleDarkMode() {
    document.body.classList.toggle("dark");
    // Save preference to localStorage
    const darkEnabled = document.body.classList.contains("dark");
    localStorage.setItem("darkMode", darkEnabled.toString());
  }

  onMount(() => {
    // Load the dark mode preference if exists
    if (localStorage.getItem("darkMode") === "true") {
      document.body.classList.add("dark");
    }
    fetchQuestions();
  });

  onDestroy(() => {
    clearInterval(timerId);
  });
</script>

<main class="container">
  <header class="header">
    <!-- Centered time text -->
    <p class="time">Time left: {formattedTime}</p>
    <!-- Dark mode toggle on the upper right -->
    <button class="toggle-btn" on:click={toggleDarkMode} aria-label="Toggle dark mode">
      <svg viewBox="0 0 24 24">
        <path d="M21 12.79A9 9 0 0112.21 3a7 7 0 000 14 9 9 0 008.79-4.21z" />
      </svg>
    </button>
  </header>

  {#if isLoading}
    <p>Loading questions...</p>
  {:else}
    {#if questions.length > 0}
      <p>{questions[currentQuestionIndex].question}</p>

      {#each questions[currentQuestionIndex].options as option}
        <button on:click={() => (selectedAnswer = option)} class:selected={selectedAnswer === option}>
          {option}
        </button>
      {/each}

      <!-- Submission button disabled if already submitted for this question -->
      <button on:click={() => submitAnswer()} disabled={hasSubmitted}>
        Submit
      </button>

      {#if examFinished}
        <p>Final score: {score}</p>
      {/if}
    {:else}
      <p>No questions available.</p>
    {/if}
  {/if}
</main>

<style>
  main {
    display: flex;
    flex-direction: column;
    align-items: center;
    max-width: 800px;
    margin: 2rem auto;
    padding: 2rem;
    background: var(--container-bg);
    border-radius: 8px;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
    font-family: "Inter", sans-serif;
  }

  /* Updated header layout */
  .header {
    position: relative;
    width: 100%;
    text-align: center;
    margin-bottom: 1rem;
  }

  .time {
    font-size: 1.25rem;
    color: var(--text);
    margin: 0; /* Remove extra margins */
  }

  .toggle-btn {
    position: absolute;
    top: 0;
    right: 0;
    background: transparent;
    border: none;
    padding: 4px;
    width: 64px;
    height: 64px;
    cursor: pointer;
  }

  .toggle-btn svg {
    width: 100%;
    height: auto;
    fill: var(--text);
  }

  p {
    font-size: 1.125rem;
    color: var(--text);
  }

  button {
    background-color: var(--primary);
    color: var(--text);
    border: 2px solid var(--secondary);
    border-radius: 5px;
    padding: 0.75rem 1.5rem;
    margin: 0.5rem;
    font-size: 1rem;
    cursor: pointer;
    transition: background-color 0.3s ease, border-color 0.3s ease;
  }

  button:hover:not(:disabled) {
    background-color: var(--secondary);
  }

  button:disabled {
    background-color: #e5e7eb;
    cursor: not-allowed;
  }

  .selected {
    background-color: var(--accent);
    color: var(--text);
    border: 2px solid transparent;
  }

  @media (max-width: 600px) {
    main {
      padding: 1.5rem;
    }
    p,
    button {
      font-size: 1rem;
    }
  }
</style>