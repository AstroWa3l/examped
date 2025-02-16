<script lang="ts">
  import { onMount, onDestroy } from "svelte";

  /** @type {{ question: string, options: string[], answer: string }[]} */
  let questions: { question: string, options: string[], answer: string }[] = [];
  let currentQuestionIndex = 0;
  /** @type {string | null} */
  let selectedAnswer: string | null = null;
  let isLoading = true;
  let score = 0;
  let hasSubmitted = false;
  let examFinished = false;
  let timeLeft = 3600; // one hour in seconds
  /** @type {number | undefined} */
  let timerId: number | undefined;

  // Reactive declaration to format timeLeft in mm:ss
  $: formattedTime = `${Math.floor(timeLeft / 60)}:${(timeLeft % 60)
    .toString()
    .padStart(2, "0")}`;

  // Reactive declaration to show current question number
  $: questionNumber = `Question ${currentQuestionIndex + 1} / ${questions.length}`;

  // Function to shuffle an array
  function shuffle<T>(array: T[]): T[] {
    for (let i = array.length - 1; i > 0; i--) {
      const j = Math.floor(Math.random() * (i + 1));
      [array[i], array[j]] = [array[j], array[i]];
    }
    return array;
  }

  async function fetchQuestions() {
    const res = await fetch("/questions.json");
    const data: { question: string, options: string[], answer: string }[] = await res.json();
    questions = shuffle(data);
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

  function resetQuiz() {
    currentQuestionIndex = 0;
    selectedAnswer = null;
    score = 0;
    hasSubmitted = false;
    examFinished = false;
    timeLeft = 3600;
    fetchQuestions();
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
    <!-- Reset button on the upper left -->
    <button class="reset-btn" on:click={resetQuiz} aria-label="Reset quiz">
      <svg viewBox="0 0 24 24">
        <path d="M12 2V6C7.58 6 4 9.58 4 14C4 18.42 7.58 22 12 22C16.42 22 20 18.42 20 14H18C18 17.31 15.31 20 12 20C8.69 20 6 17.31 6 14C6 10.69 8.69 8 12 8V12L17 7L12 2Z" />
      </svg>
    </button>
    <!-- Centered time text -->
    <p class="time">Time left: {formattedTime}</p>
    <!-- Current question number -->
    <p class="question-number">{questionNumber}</p>
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

      <!-- Reset button to start over -->
      <button on:click={resetQuiz}>
        Reset
      </button>
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
    padding: 1rem;
  }

  .reset-btn {
    position: absolute;
    top: 0;
    left: 0;
    background: transparent;
    border: none;
    padding: 4px;
    width: 64px;
    height: 64px;
    cursor: pointer;
  }

  .reset-btn svg {
    width: 100%;
    height: auto;
    fill: var(--text);
  }

  .time {
    font-size: 1.25rem;
    color: var(--text);
    margin: 0; /* Remove extra margins */
  }

  .question-number {
    font-size: 1.25rem;
    color: var(--text);
    margin: 0.5rem 0;
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
    margin-top: 10px;
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