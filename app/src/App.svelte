<script>
  import birds20 from './lib/data/birds-20.json'
  import birds50 from './lib/data/birds-50.json'
  import birdsAll from './lib/data/birds-all.json'

  const difficulties = [
    { id: 'easy', label: 'Let', detail: '20 fugle', birds: birds20 },
    { id: 'medium', label: 'Mellem', detail: '50 fugle', birds: birds50 },
    { id: 'hard', label: 'Svær', detail: 'Alle fugle', birds: birdsAll },
  ]

  let selectedDifficulty = difficulties[0]
  let screen = 'splash'
  let answerPool = []
  let currentBird = null
  let wrongGuesses = new Set()
  let correctGuess = null
  let guessesThisRound = 0
  let score = []
  let showStopDialog = false
  let showCorrectToast = false
  let showWrongToast = false
  let showGiveUpDialog = false
  let correctToastTimeout
  let wrongToastTimeout
  let audioElement

  $: gridBirds = selectedDifficulty.birds
  $: toastMessage = showCorrectToast ? 'Korrekt!' : showWrongToast ? 'forkert' : ''
  $: toastTone = showCorrectToast ? 'correct' : showWrongToast ? 'wrong' : 'idle'

  function shuffle(items) {
    return [...items].sort(() => Math.random() - 0.5)
  }

  function chooseBird(pool) {
    return pool[Math.floor(Math.random() * pool.length)]
  }

  function playCurrentSong() {
    if (!audioElement || !currentBird) return

    audioElement.pause()
    audioElement.currentTime = 0
    audioElement.play().catch(() => {
      // Browseren kan afvise autoplay, indtil brugeren trykker på afspil.
    })
  }

  function nextRound(pool = answerPool) {
    const refreshedPool = pool.length ? pool : shuffle(selectedDifficulty.birds)
    answerPool = refreshedPool
    currentBird = chooseBird(refreshedPool)
    wrongGuesses = new Set()
    correctGuess = null
    guessesThisRound = 0

    setTimeout(playCurrentSong)
  }

  function startGame() {
    clearTimeout(correctToastTimeout)
    clearTimeout(wrongToastTimeout)
    score = []
    showStopDialog = false
    showCorrectToast = false
    showWrongToast = false
    showGiveUpDialog = false
    correctGuess = null
    screen = 'game'
    nextRound(shuffle(selectedDifficulty.birds))
  }

  function guessBird(bird) {
    if (
      !currentBird ||
      showCorrectToast ||
      showGiveUpDialog ||
      wrongGuesses.has(bird.title)
    ) return

    clearTimeout(wrongToastTimeout)
    showWrongToast = false
    guessesThisRound += 1

    if (bird.title !== currentBird.title) {
      wrongGuesses = new Set([...wrongGuesses, bird.title])
      showWrongToast = true
      clearTimeout(wrongToastTimeout)
      wrongToastTimeout = setTimeout(() => {
        showWrongToast = false
      }, 2000)
      return
    }

    score = [
      { bird: currentBird.title, guesses: guessesThisRound },
      ...score,
    ]

    const remaining = answerPool.filter((item) => item.title !== currentBird.title)
    const nextPool = remaining.length ? remaining : shuffle(selectedDifficulty.birds)

    audioElement?.pause()
    correctGuess = bird.title
    showCorrectToast = true
    clearTimeout(correctToastTimeout)
    correctToastTimeout = setTimeout(() => {
      showCorrectToast = false
      correctGuess = null
      nextRound(nextPool)
    }, 2000)
  }

  function giveUp() {
    if (!currentBird || showCorrectToast) return

    audioElement?.pause()
    showGiveUpDialog = true
  }

  function continueAfterGiveUp() {
    const remaining = answerPool.filter((item) => item.title !== currentBird.title)
    const nextPool = remaining.length ? remaining : shuffle(selectedDifficulty.birds)

    showGiveUpDialog = false
    nextRound(nextPool)
  }

  function requestStop() {
    if (screen === 'game') {
      showStopDialog = true
    }
  }

  function stopGame() {
    clearTimeout(correctToastTimeout)
    clearTimeout(wrongToastTimeout)
    audioElement?.pause()
    showStopDialog = false
    showCorrectToast = false
    showWrongToast = false
    showGiveUpDialog = false
    screen = 'splash'
    currentBird = null
    answerPool = []
    wrongGuesses = new Set()
    correctGuess = null
    guessesThisRound = 0
  }

  function handleKeydown(event) {
    if (showCorrectToast || showWrongToast || showGiveUpDialog) return

    if (event.key === 'Escape') {
      if (showStopDialog) {
        showStopDialog = false
      } else {
        requestStop()
      }
    }
  }
</script>

<svelte:window onkeydown={handleKeydown} />

{#if screen === 'splash'}
  <main class="splash">
    <section class="intro">
      <p class="kicker">Lyt, kig, gæt</p>
      <h1>Gæt en fugl</h1>
      <p>
        Hør en fuglesang, og find fuglen i billedgitteret. 
      </p>
    </section>

    <section class="difficulty-panel" aria-label="Vælg sværhedsgrad">
      <div class="difficulty-buttons">
        {#each difficulties as difficulty}
          <button
            class:active={selectedDifficulty.id === difficulty.id}
            type="button"
            onclick={() => (selectedDifficulty = difficulty)}
          >
            <span>{difficulty.label}</span>
            <small>{difficulty.detail}</small>
          </button>
        {/each}
      </div>

      <button class="start-button" type="button" onclick={startGame}>Start</button>
    </section>
  </main>
{:else}
  <main class="game">
    <section class="play-area">
      <header class="game-header">
        <div>
          <p class="kicker">{selectedDifficulty.label}</p>
          <h1>Hvilken fugl synger?</h1>
        </div>
        <div class="header-actions">
          <button class="secondary-button" type="button" onclick={giveUp}>Opgiv</button>
          <button class="secondary-button" type="button" onclick={requestStop}>Stop</button>
          <div
            class="header-toast"
            class:correct={toastTone === 'correct'}
            class:wrong={toastTone === 'wrong'}
            aria-live="polite"
            aria-label={toastMessage || 'Ingen besked'}
          >
            {toastMessage}
          </div>
        </div>
      </header>

      <audio
        bind:this={audioElement}
        src={currentBird ? `sounds/${currentBird.wav}` : ''}
        autoplay
        loop
        onloadedmetadata={playCurrentSong}
      >
        Din browser kan ikke afspille lyden.
      </audio>

      <p class="round-status">{guessesThisRound} gæt i denne runde</p>

      <div class="bird-grid">
        {#each gridBirds as bird}
          <button
            class="bird-tile"
            class:muted={wrongGuesses.has(bird.title)}
            class:correct={correctGuess === bird.title}
            type="button"
            disabled={wrongGuesses.has(bird.title)}
            onclick={() => guessBird(bird)}
          >
            <img src={`thumbs/${bird.image}`} alt={bird.title} loading="lazy" />
            <span>{bird.title}</span>
          </button>
        {/each}
      </div>
    </section>

    <aside class="score-panel" aria-label="Scoreliste">
      <h2>Score</h2>
      {#if score.length === 0}
        <p class="empty-score">Ingen rigtige gæt endnu.</p>
      {:else}
        <ol>
          {#each score as item}
            <li>
              <span>{item.bird}</span>
              <strong>{item.guesses}</strong>
            </li>
          {/each}
        </ol>
      {/if}
    </aside>
  </main>
{/if}

{#if showGiveUpDialog && currentBird}
  <div class="dialog-backdrop" role="presentation">
    <div
      class="answer-dialog"
      role="dialog"
      aria-modal="true"
      aria-labelledby="answer-title"
    >
      <p class="kicker">Det var</p>
      <h2 id="answer-title">{currentBird.title}</h2>
      <img src={`thumbs/${currentBird.image}`} alt={currentBird.title} />
      <button class="start-button" type="button" onclick={continueAfterGiveUp}>
        Næste fugl
      </button>
    </div>
  </div>
{/if}

{#if showStopDialog}
  <div class="dialog-backdrop" role="presentation">
    <div
      class="stop-dialog"
      role="dialog"
      aria-modal="true"
      aria-labelledby="stop-title"
    >
      <h2 id="stop-title">stoppe spillet?</h2>
      <div>
        <button class="danger-button" type="button" onclick={stopGame}>ja</button>
        <button type="button" onclick={() => (showStopDialog = false)}>nej</button>
      </div>
    </div>
  </div>
{/if}
