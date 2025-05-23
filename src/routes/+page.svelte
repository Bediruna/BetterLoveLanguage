<script>
  import { onMount } from "svelte";
  import GitHubIcon from "../components/GitHubIcon.svelte";

  let scenarios = [];
  let descriptions = [];
  let currentScenarios = [];
  let selectedAnswers = [];
  let usedScenarioIds = new Set();
  let questionIndex = 1;
  let quizComplete = false;
  let loveLanguageCounts = {};
  let sortedLoveLanguages = [];
  let totalSelections = 0;
  let totalQuestions = 25; // Only ask 25 questions
  let topLoveLanguage = null;

  // Mapping of love languages to Bootstrap color classes
  const loveLanguageColors = {
    "Words of Affirmation": "bg-primary",
    "Acts of Service": "bg-success",
    "Receiving Gifts": "bg-info",
    "Quality Time": "bg-warning",
    "Physical Touch": "bg-danger",
  };

  // Function to get the Bootstrap color class for a love language
  function getProgressBarClass(language) {
    return loveLanguageColors[language] || "bg-secondary";
  }

  // Fisher-Yates shuffle algorithm
  function shuffleArray(array) {
    for (let i = array.length - 1; i > 0; i--) {
      const j = Math.floor(Math.random() * (i + 1));
      [array[i], array[j]] = [array[j], array[i]];
    }
  }

  // Fetch scenarios and descriptions
  onMount(async () => {
    try {
      const scenarioResponse = await fetch("/data/scenarios.json");
      scenarios = await scenarioResponse.json();

      const descriptionResponse = await fetch("/data/descriptions.json");
      descriptions = await descriptionResponse.json();

      if (scenarios.length === 0 || descriptions.length === 0) {
        console.error("Scenarios or descriptions data is empty.");
        return;
      }

      shuffleArray(scenarios);

      loadNextQuestion();
    } catch (error) {
      console.error("Error fetching data:", error);
    }
  });

  // Load the next question by selecting two scenarios of the same weight
  function loadNextQuestion() {
    if (
      questionIndex > totalQuestions ||
      usedScenarioIds.size >= scenarios.length
    ) {
      quizComplete = true;
      calculateResults();
    } else {
      // Get available scenarios that haven't been used
      const availableScenarios = scenarios.filter(
        (s) => !usedScenarioIds.has(s.id),
      );

      // Look for a pair of scenarios that share the same weight
      let foundPair = false;
      let scenarioPair = [];
      for (let i = 0; i < availableScenarios.length; i++) {
        const scenario1 = availableScenarios[i];
        // Get all scenarios (other than scenario1) with the same weight
        const sameWeightScenarios = availableScenarios.filter(
          (s) => s.Weight === scenario1.Weight && s.id !== scenario1.id,
        );
        if (sameWeightScenarios.length > 0) {
          // Prefer a scenario with a different love language if possible
          let scenario2 = sameWeightScenarios.find(
            (s) => s.Language !== scenario1.Language,
          );
          if (!scenario2) {
            scenario2 = sameWeightScenarios[0];
          }
          scenarioPair = [scenario1, scenario2];
          foundPair = true;
          break;
        }
      }

      if (!foundPair) {
        quizComplete = true;
        calculateResults();
        return;
      }

      // Mark both scenarios as used and update currentScenarios
      usedScenarioIds.add(scenarioPair[0].id);
      usedScenarioIds.add(scenarioPair[1].id);
      currentScenarios = scenarioPair;
    }
  }

  // Handle answer selection
  function selectScenario(selected) {
    selectedAnswers.push(selected);
    totalSelections++;
    questionIndex++;
    loadNextQuestion();
  }

  // Calculate quiz results
  function calculateResults() {
    loveLanguageCounts = {};

    selectedAnswers.forEach((answer) => {
      const language = answer.Language;
      loveLanguageCounts[language] = (loveLanguageCounts[language] || 0) + 1;
    });

    let maxPercentage = 0;

    // Convert counts to percentages
    for (let language in loveLanguageCounts) {
      const percentage = Math.round(
        (loveLanguageCounts[language] / totalSelections) * 100,
      );
      loveLanguageCounts[language] = percentage;

      if (percentage > maxPercentage) {
        maxPercentage = percentage;
        topLoveLanguage = language;
      }
    }

    // Create a sorted array of love languages based on percentages
    sortedLoveLanguages = Object.entries(loveLanguageCounts).sort(
      (a, b) => b[1] - a[1],
    );
  }

  // Retrieve love language description
  function getLoveLanguageDescription(language) {
    return descriptions.find((desc) => desc.language === language);
  }
</script>

<main class="container">
  {#if !quizComplete && currentScenarios.length === 2}
    <div class="quiz-content">
      <h1 class="text-center mb-4">I feel most loved when...</h1>
      <p class="text-center">Question {questionIndex} of {totalQuestions}</p>

      <!-- Fixed width buttons container -->
      <div class="quiz-buttons-container">
        <button
          class="btn btn-primary btn-lg quiz-button"
          on:click={() => selectScenario(currentScenarios[0])}
        >
          {currentScenarios[0].Scenario}
        </button>

        <button
          class="btn btn-secondary btn-lg quiz-button"
          on:click={() => selectScenario(currentScenarios[1])}
        >
          {currentScenarios[1].Scenario}
        </button>
      </div>
    </div>
  {:else if quizComplete}
    <br />
    <h4 class="text-center">
      You've completed the quiz! Here are your results:
    </h4>
    <div class="progress-container mt-4">
      {#each sortedLoveLanguages as [language, percentage]}
        <div class="mb-4">
          <div class="d-flex justify-content-between">
            <span><strong>{language}</strong></span>
            <span>{percentage}%</span>
          </div>
          <div class="progress">
            <div
              class="progress-bar {getProgressBarClass(language)}"
              role="progressbar"
              style="width: {percentage}%"
              aria-valuenow={percentage}
              aria-valuemin="0"
              aria-valuemax="100"
            ></div>
          </div>
        </div>
      {/each}
    </div>

    {#if topLoveLanguage}
      <div class="top-love-language mt-4 p-3">
        <h2 class="text-center">
          Your top love language is: <strong>{topLoveLanguage}</strong>
        </h2>
        {#if getLoveLanguageDescription(topLoveLanguage)}
          <p>
            <strong>Description:</strong>
            {getLoveLanguageDescription(topLoveLanguage).description}
          </p>
          <p>
            <strong>How to Communicate:</strong>
            {getLoveLanguageDescription(topLoveLanguage).howToCommunicate}
          </p>
          <p>
            <strong>Actions to Take:</strong>
            {getLoveLanguageDescription(topLoveLanguage).actionsToTake}
          </p>
          <p>
            <strong>Things to Avoid:</strong>
            {getLoveLanguageDescription(topLoveLanguage).thingsToAvoid}
          </p>
        {/if}
      </div>
    {/if}
  {:else}
    <p class="text-center">Loading...</p>
  {/if}
</main>
<footer class="footer-text">
  <a href="https://github.com/Bediruna/better-love-language" target="_blank">
    <GitHubIcon />
  </a>
</footer>

<style>
  .quiz-container {
    margin-top: 20px;
  }

  /* New button container styling */
  .quiz-buttons-container {
    display: flex;
    flex-direction: column;
    gap: 20px;
    width: 100%;
    max-width: 800px; /* Increased from 600px to allow wider buttons */
    margin: 0 auto;
  }

  /* Button styling */
  .quiz-button {
    width: 100%;
    min-height: 120px; /* Changed from fixed height to minimum height */
    height: auto; /* Allow the height to grow */
    padding: 15px;
    font-size: 18px;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    text-align: center;
    white-space: normal; /* Ensure text wraps */
    word-wrap: break-word; /* Better handling of long words */
  }

  .footer-text {
    position: fixed;
    bottom: 10px;
    right: 10px;
    font-size: 14px;
    color: #555;
  }

  .top-love-language {
    background-color: #f9f9f9;
    border: 1px solid #ddd;
    border-radius: 5px;
    padding: 20px;
  }

  .progress-container {
    max-width: 600px;
    margin: 0 auto;
  }

  .progress {
    height: 25px;
  }

  .progress-bar {
    font-weight: bold;
    line-height: 25px;
  }

  /* Quiz content styling */
  .quiz-content {
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    min-height: 80vh;
    width: 100%;
  }

  /* Media query for larger screens */
  @media (min-width: 768px) {
    .quiz-buttons-container {
      flex-direction: row;
      max-width: 800px; /* Increased container width */
    }

    .quiz-button {
      width: 380px; /* Increased fixed width for buttons on larger screens */
    }
  }
</style>
