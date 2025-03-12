<!-- App.vue -->
<template>
  <div class="app">
    <div class="header">
      <h1>Who Liked? 🎮</h1>
    </div>

    <!-- Écran de connexion -->
    <div v-if="!isConnected" class="login-screen">
      <div class="login-card">
        <h2>Connexion</h2>
        <input v-model="userId" placeholder="User ID" />
        <input v-model="token" placeholder="Token" type="password" />
        <button @click="connect">Se connecter</button>
      </div>
    </div>

    <!-- Écran principal -->
    <div v-else class="game-screen">
      <!-- Pas dans une partie -->
      <div v-if="!currentGame" class="game-menu">
        <div class="menu-card">
          <h2>Menu Principal</h2>
          <button class="primary-btn" @click="createGame">Créer une partie</button>
          <div class="separator">ou</div>
          <div class="join-game">
            <input v-model="gamePin" placeholder="Code PIN" maxlength="6" />
            <button @click="joinGame">Rejoindre</button>
          </div>
        </div>
      </div>

      <!-- Dans une partie -->
      <div v-else class="game-container">
        <!-- En attente -->
        <div v-if="gameStatus === 'waiting' || gameStatus === 'ready'" class="waiting-room">
          <div class="waiting-card">
            <h2>Salle d'attente</h2>
            <div class="game-info">
              <div class="pin-display">
                <span class="label">PIN</span>
                <span class="value">{{ currentGame.pin }}</span>
              </div>
              <div class="players-count">
                <span class="label">Joueurs</span>
                <span class="value">{{ players.length }}/2</span>
              </div>
            </div>
            <div class="players-list">
              <div v-for="player in players" :key="player.id" class="player">
                <span class="player-name">{{ formatPlayerName(player.nickname) }}</span>
                <span v-if="player.ready" class="ready">✅ Prêt</span>
                <span v-else class="not-ready">⏳ En attente</span>
              </div>
            </div>
            <div class="action-buttons">
              <button v-if="!isReady" class="primary-btn" @click="markReady">Je suis prêt !</button>
              <button 
                v-else-if="isCreator && gameStatus === 'ready'" 
                class="start-btn" 
                @click="startGame"
              >
                Démarrer la partie
              </button>
              <button v-else-if="gameStatus === 'ready'" disabled>En attente du créateur...</button>
              <button v-else disabled>En attente des autres joueurs...</button>
            </div>
          </div>
        </div>

        <!-- En jeu -->
        <div v-else-if="gameStatus === 'in_progress'" class="game-play">
          <div class="game-header">
            <div class="progress-info">
              <div class="round-info">
                <span class="label">Round</span>
                <span class="value">{{ currentRound }}/3</span>
              </div>
              <div class="video-info">
                <span class="label">Vidéo</span>
                <span class="value">{{ currentVideo }}/{{ currentGame.videosPerRound }}</span>
              </div>
            </div>
            <div class="scores-panel">
              <div v-for="player in players" :key="player.id" class="player-score">
                <div class="score-header">
                  <span class="player-name">{{ formatPlayerName(player.nickname) }}</span>
                  <div class="player-badges">
                    <span v-if="player.streak >= 3" class="combo-badge" :title="`Combo actif: +5 points par bonne réponse`">
                      🔥 x{{ player.streak }}
                    </span>
                  </div>
                </div>
                <div class="score-value">{{ player.score }} pts</div>
              </div>
            </div>
          </div>

          <!-- Zone de jeu -->
          <div class="game-area">
            <!-- Résumé du round -->
            <div v-if="showRoundSummary && roundSummary" class="round-summary">
              <div class="summary-card">
                <div class="round-header">
                  <div class="round-badge">Round {{ roundSummary.roundNumber }}</div>
                  <h2>🏆 Résultats du Round</h2>
                </div>
                
                <div class="round-stats">
                  <div class="stat-box">
                    <div class="stat-icon">🎯</div>
                    <div class="stat-value">{{ getTotalCorrectVotes() }}/{{ getTotalVotes() }}</div>
                    <div class="stat-label">Votes corrects</div>
                  </div>
                  <div class="stat-box">
                    <div class="stat-icon">⚡</div>
                    <div class="stat-value">{{ getTotalPoints() }}</div>
                    <div class="stat-label">Points gagnés</div>
                  </div>
                  <div class="stat-box">
                    <div class="stat-icon">📊</div>
                    <div class="stat-value">{{ Math.round(getAverageAccuracy()) }}%</div>
                    <div class="stat-label">Précision moyenne</div>
                  </div>
                </div>
                
                <div v-if="roundSummary.bestPlayer" class="best-player">
                  <div class="trophy">👑</div>
                  <div class="best-player-info">
                    <div class="best-player-title">Meilleur joueur du round</div>
                    <div class="best-player-name">{{ getPlayerNickname(roundSummary.bestPlayer) }}</div>
                    <div class="best-player-score">{{ roundSummary.bestScore }} points</div>
                  </div>
                </div>
                
                <div class="player-stats-grid">
                  <div v-for="player in players" :key="player.id" class="player-round-stats">
                    <div class="player-round-header">
                      <span class="player-name">{{ formatPlayerName(player.nickname) }}</span>
                      <div class="player-badges">
                        <span v-if="roundSummary.bestPlayer && player.id === roundSummary.bestPlayer" class="winner-badge">
                          🏅 Meilleur joueur
                        </span>
                        <span v-if="getPlayerCorrectVotes(player.id) === Math.max(...Object.values(roundSummary.playerCorrectVotes))" class="bonus-badge">
                          ⭐ +5 pts bonus
                        </span>
                      </div>
                    </div>
                    <div class="stats-row">
                      <div class="stat-item">
                        <div class="stat-value">{{ getPlayerScore(player.id) }}</div>
                        <div class="stat-label">Points</div>
                      </div>
                      <div class="stat-item">
                        <div class="stat-value">{{ getPlayerCorrectVotes(player.id) }}/{{ getPlayerTotalVotes(player.id) }}</div>
                        <div class="stat-label">Votes corrects</div>
                      </div>
                      <div class="stat-item">
                        <div class="stat-value">{{ Math.round(getPlayerAccuracy(player.id)) }}%</div>
                        <div class="stat-label">Précision</div>
                      </div>
                    </div>
                    <div class="player-progress">
                      <div class="progress-bar-label">Progression</div>
                      <div class="progress-bar">
                        <div class="progress" :style="{ width: `${Math.round(getPlayerAccuracy(player.id))}%` }" :class="{ 'high-accuracy': getPlayerAccuracy(player.id) > 70 }"></div>
                      </div>
                    </div>
                  </div>
                </div>
                
                <div class="next-round-info">
                  <p v-if="roundSummary.roundNumber < 3">Préparation du Round {{ roundSummary.roundNumber + 1 }}...</p>
                  <p v-else>Préparation des résultats finaux...</p>
                  <div class="progress-bar">
                    <div class="progress" :style="{ width: `${roundTransitionProgress}%` }"></div>
                  </div>
                </div>
              </div>
            </div>
            
            <!-- Écran de jeu principal -->
            <div v-else>
              <transition name="fade" mode="out-in">
                <!-- Phase de vote ou écran propriétaire -->
                <div v-if="currentVideoData && !hasVoted && !showResults" key="voting">
                  <!-- Écran spécial pour le propriétaire de la vidéo -->
                  <div v-if="isVideoOwner && currentVideoData" class="video-owner-screen">
                    <div class="video-frame">
                      <img :src="currentVideoData.cover" alt="Video cover" class="video-cover-owner" />
                      <div class="emoji-overlay">
                        <span class="emoji">🤫</span>
                      </div>
                    </div>
                    <div class="owner-message">
                      <h2 class="owner-title">PRETEND YOU DON'T</h2>
                      <h2 class="owner-title">TO EARN POINTS</h2>
                      <p class="owner-subtitle">Yes dirty boy... you've liked this video</p>
                    </div>
                    <div class="owner-actions">
                      <p class="secret-text">Keep this secret?</p>
                      <div class="action-bar">
                        <button 
                          @click="skipReveal" 
                          class="skip-button" 
                          :disabled="hasSkippedReveal || skipsUsedThisRound > 0"
                          :title="skipsUsedThisRound > 0 ? 'Déjà utilisé ce round' : ''"
                        >
                          {{ hasSkippedReveal ? 'Identité masquée (-5 pts)' : 'Masquer mon identité (-5 pts)' }}
                          <span class="limit-text">(1 fois par round)</span>
                        </button>
                        <span class="counter">{{ remainingVideos }} left</span>
                      </div>
                    </div>
                  </div>

                  <!-- Affichage normal pour les autres joueurs -->
                  <div v-else-if="currentVideoData" class="video-display">
                    <div class="video-container">
                      <img :src="currentVideoData.cover" alt="Video cover" class="video-cover" />
                      <div class="video-stats">
                        <div class="stat">
                          <span class="icon">👤</span>
                          <span class="value">{{ formatPlayerName(currentVideoData.author.nickname) }}</span>
                        </div>
                        <div class="stat">
                          <span class="icon">❤️</span>
                          <span class="value">{{ formatNumber(currentVideoData.stats.diggCount) }}</span>
                        </div>
                        <div class="stat">
                          <span class="icon">👁️</span>
                          <span class="value">{{ formatNumber(currentVideoData.stats.playCount) }}</span>
                        </div>
                      </div>
                    </div>
                    <div class="voting-section">
                      <h3>Qui a liké cette vidéo ?</h3>
                      <div class="votes-counter">
                        <span>{{ votesProgress }} votes</span>
                        <div class="votes-progress-bar">
                          <div 
                            class="votes-progress" 
                            :style="{ width: `${(Object.keys(currentVideoData?.votes || {}).length / votablePlayers.length) * 100}%` }"
                          ></div>
                        </div>
                      </div>
                      <div class="vote-buttons">
                        <button 
                          v-for="player in votablePlayers" 
                          :key="player.id"
                          @click="submitVote(player.id)"
                          :disabled="hasVoted"
                          :class="{ voted: hasVoted }"
                        >
                          {{ formatPlayerName(player.nickname) }}
                        </button>
                      </div>
                    </div>
                  </div>
                </div>

                <!-- Phase de révélation -->
                <div v-else-if="hasVoted && !showResults && videoResults && currentVideoData" key="reveal" class="reveal-screen">
                  <div class="reveal-card">
                    <div class="reveal-header">
                      <h2>🎭 Révélation</h2>
                    </div>
                    <div class="reveal-content" :class="{ 'blurred': videoResults.skipReveal }">
                      <div class="video-owner-reveal">
                        <img :src="currentVideoData.cover" alt="Video cover" class="video-cover-reveal" />
                        <div class="owner-info">
                          <p class="reveal-text">Cette vidéo a été likée par...</p>
                          <h3 class="owner-name" :class="{ 'hidden': videoResults.skipReveal }">
                            {{ getPlayerNickname(videoResults.contributedBy) }}
                          </h3>
                          <p v-if="videoResults.skipReveal" class="skipped-text">
                            Le propriétaire a choisi de rester anonyme
                          </p>
                        </div>
                      </div>
                    </div>
                  </div>
                </div>

                <!-- Phase de résultats -->
                <div v-else-if="showResults && videoResults" key="results" class="results">
                  <div class="results-card">
                    <h3>Résultats du vote</h3>
                    <div class="result-content">
                      <div class="video-owner" :class="{ 'blurred': videoResults.skipReveal }">
                        <span class="label">Vidéo likée par</span>
                        <span v-if="!videoResults.skipReveal" class="value">
                          {{ getPlayerNickname(videoResults.contributedBy) }}
                        </span>
                        <span v-else class="value anonymous">Anonyme</span>
                      </div>
                      <div class="votes-summary">
                        <div v-for="(vote, playerId) in videoResults.votes" :key="playerId" class="vote">
                          <div class="vote-player">
                            <span class="player-name">{{ getPlayerNickname(playerId) }}</span>
                            <div class="vote-details">
                              <span :class="['vote-result', vote.correct ? 'correct' : 'incorrect']">
                                {{ vote.correct ? '✅' : '❌' }}
                              </span>
                              <div v-if="vote.correct" class="points-breakdown">
                                <span class="base-points">+10</span>
                                <span v-if="vote.points > 10" class="combo-points">+5</span>
                              </div>
                            </div>
                          </div>
                          <div class="vote-points">
                            <span class="points">{{ vote.points }}</span>
                            <span class="unit">pts</span>
                          </div>
                        </div>
                      </div>
                    </div>
                  </div>
                </div>
              </transition>
            </div>
          </div>
        </div>

        <!-- Fin de partie -->
        <div v-else-if="gameStatus === 'completed'" class="game-end">
          <div class="end-card">
            <h2>🎉 Partie terminée !</h2>
            <div class="final-scores">
              <div v-for="(player, index) in playersByScore" :key="player.id" class="final-score">
                <div class="rank">{{ index + 1 }}</div>
                <div class="player-info">
                  <span class="player-name">{{ formatPlayerName(player.nickname) }}</span>
                  <span class="final-points">{{ player.score }} points</span>
                </div>
              </div>
            </div>
            <button class="primary-btn" @click="leaveGame">Retour au menu</button>
          </div>
        </div>

        <!-- Écran de chargement -->
        <div v-if="isLoading" class="loading-screen">
          <div class="loading-spinner">
            <div class="spinner"></div>
            <p>Chargement de la prochaine vidéo...</p>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import io from 'socket.io-client';

export default {
  name: 'App',
  data() {
    return {
      // Configuration
      SERVER_URL: 'http://localhost:5400',
      NAMESPACE: '/game',

      // État de connexion
      isConnected: false,
      socket: null,
      userId: '',
      token: '',

      // État du jeu
      currentGame: null,
      gameStatus: 'waiting',
      gamePin: '',
      players: [],
      isReady: false,

      // État de la partie
      currentRound: 1,
      currentVideo: 1,
      currentVideoData: null,
      hasVoted: false,
      showResults: false,
      videoResults: null,
      isVideoOwner: false,

      // État du résumé du round
      showRoundSummary: false,
      roundSummary: null,
      roundTransitionProgress: 0,
      roundTransitionInterval: null,
      showReveal: false,
      hasSkippedReveal: false,
      isLoading: false,
      skipsUsedThisRound: 0,
    };
  },

  computed: {
    votablePlayers() {
      if (!this.currentVideoData) return this.players.filter(p => p.id !== this.userId);
      
      // Si je suis le propriétaire de la vidéo, je ne devrais pas voir d'options de vote
      // (mais ce cas est déjà géré par l'affichage conditionnel de l'écran spécial)
      
      // Pour les autres joueurs, ils peuvent voter pour tous les joueurs sauf eux-mêmes
      return this.players.filter(p => p.id !== this.userId);
    },

    playersByScore() {
      return [...this.players].sort((a, b) => b.score - a.score);
    },

    remainingVideos() {
      if (!this.currentGame || !this.currentRound || !this.currentVideo || !this.currentGame.videosPerRound) {
        return 0;
      }
      return this.currentGame.videosPerRound - this.currentVideo;
    },

    isCreator() {
      return this.players.find(p => p.id === this.userId)?.isCreator || false;
    },
    
    allPlayersReady() {
      return this.players.length >= 2 && this.players.every(p => p.ready);
    },

    allPlayersVoted() {
      if (!this.currentVideoData || !this.currentVideoData.votes) return false;
      const votablePlayersCount = this.votablePlayers.length;
      const votesCount = Object.keys(this.currentVideoData.votes).length;
      return votesCount >= votablePlayersCount;
    },

    votesProgress() {
      if (!this.currentVideoData || !this.currentVideoData.votes) return '0 / 0';
      const votablePlayersCount = this.votablePlayers.length;
      const votesCount = Object.keys(this.currentVideoData.votes).length;
      return `${votesCount} / ${votablePlayersCount}`;
    }
  },

  methods: {
    // Connexion
    connect() {
      if (!this.userId || !this.token) return;

      this.socket = io(`${this.SERVER_URL}${this.NAMESPACE}`, {
        auth: { token: this.token },
        transports: ['websocket']
      });

      this.setupSocketListeners();
    },

    // Configuration des écouteurs socket
    setupSocketListeners() {
      this.socket.on('connect', () => {
        this.isConnected = true;
      });

      this.socket.on('disconnect', () => {
        this.isConnected = false;
        this.resetGameState();
      });

      this.socket.on('error', (error) => {
        console.error('Socket error:', error);
        alert(error.message);
      });

      // Événements de jeu
      this.socket.on('gameStatus', this.handleGameStatus);
      this.socket.on('gameCreated', this.handleGameCreated);
      this.socket.on('gameJoined', this.handleGameJoined);
      this.socket.on('gameReady', this.handleGameReady);
      this.socket.on('gameStarted', this.handleGameStarted);
      this.socket.on('videoStarted', this.handleVideoStarted);
      this.socket.on('videoResults', this.handleVideoResults);
      this.socket.on('gameCompleted', this.handleGameCompleted);
      this.socket.on('videoOwnership', this.handleVideoOwnership);

      // Nouvel écouteur pour le résumé du round
      this.socket.on('roundSummary', this.handleRoundSummary);
      this.socket.on('newRound', this.handleNewRound);

      // Nouvel écouteur pour la déduction des points
      this.socket.on('pointsDeducted', this.handlePointsDeducted);
    },

    // Actions de jeu
    createGame() {
      this.socket.emit('createGame');
    },

    joinGame() {
      if (!this.gamePin) return;
      this.socket.emit('joinGame', { pin: this.gamePin });
    },

    markReady() {
      this.socket.emit('playerReady');
    },

    submitVote(votedUserId) {
      this.socket.emit('submitVote', { votedUserId });
    },

    leaveGame() {
      this.socket.emit('leaveGame');
      this.resetGameState();
    },

    // Gestionnaires d'événements
    handleGameStatus(status) {
      console.log('Game status received:', status);
      this.gameStatus = status.status;
      this.players = status.players.details;
      this.currentRound = status.currentRound;
      this.currentVideo = status.currentVideo;

      // Mettre à jour hasVoted si on trouve notre vote dans la vidéo actuelle
      if (this.currentVideoData && this.currentVideoData.votes) {
        this.hasVoted = !!this.currentVideoData.votes[this.userId];
      }
    },

    handleGameCreated(data) {
      this.currentGame = data.game;
      this.gameStatus = 'waiting';
    },

    handleGameJoined(data) {
      this.currentGame = data.game;
      this.gameStatus = 'waiting';
    },

    handleGameReady() {
      this.isReady = true;
    },

    handleGameStarted(data) {
      this.gameStatus = 'in_progress';
      this.currentRound = 1;
      this.currentVideo = 1;
      // S'assurer que videosPerRound est toujours disponible
      if (!this.currentGame) {
        this.currentGame = {};
      }
      // Préserver la valeur de videosPerRound si elle n'est pas fournie dans data
      if (!this.currentGame.videosPerRound && data.videosPerRound) {
        this.currentGame.videosPerRound = data.videosPerRound;
      }
      console.log('Game started with videosPerRound:', data.videosPerRound);
    },

    handleVideoStarted(data) {
      this.isLoading = true;
      this.currentRound = data.roundNumber;
      this.currentVideo = data.videoNumber;
      this.currentVideoData = data.video;
      this.hasVoted = false;
      this.showResults = false;
      this.videoResults = null;
      this.isVideoOwner = false;
      this.hasSkippedReveal = false;
      this.$nextTick(() => {
        this.isLoading = false;
      });
    },

    handleVideoResults(results) {
      this.videoResults = results;
      this.hasVoted = true; // On met hasVoted à true seulement quand tous les joueurs ont voté
      
      // Afficher d'abord la révélation pendant 3 secondes
      setTimeout(() => {
        this.showResults = true;
      }, 3000);
    },

    showVoteResults() {
      this.showResults = true;
    },

    handleGameCompleted() {
      this.gameStatus = 'completed';
    },

    handleVideoOwnership(data) {
      this.isVideoOwner = data.isVideoOwner;
      console.log('Video ownership status:', this.isVideoOwner);
    },

    handleRoundSummary(summary) {
      console.log('Round summary received:', summary);
      
      // Vérifier que le résumé est valide
      if (!summary || typeof summary !== 'object') {
        console.error('Invalid round summary received:', summary);
        return;
      }
      
      // Afficher les données brutes pour le débogage
      console.log('Raw summary data:', {
        roundNumber: summary.roundNumber,
        playerScores: summary.playerScores,
        playerCorrectVotes: summary.playerCorrectVotes,
        playerTotalVotes: summary.playerTotalVotes,
        playerAccuracy: summary.playerAccuracy,
        bestPlayer: summary.bestPlayer,
        bestScore: summary.bestScore
      });
      
      // S'assurer que toutes les propriétés nécessaires sont présentes
      this.roundSummary = {
        roundNumber: summary.roundNumber || 0,
        playerScores: summary.playerScores || {},
        playerCorrectVotes: summary.playerCorrectVotes || {},
        playerTotalVotes: summary.playerTotalVotes || {},
        playerAccuracy: summary.playerAccuracy || {},
        bestPlayer: summary.bestPlayer || null,
        bestScore: summary.bestScore || 0
      };
      
      // Vérifier que les données sont correctes pour chaque joueur
      this.players.forEach(player => {
        console.log(`Player ${player.id} data:`, {
          score: this.getPlayerScore(player.id),
          correctVotes: this.getPlayerCorrectVotes(player.id),
          totalVotes: this.getPlayerTotalVotes(player.id),
          accuracy: this.getPlayerAccuracy(player.id)
        });
      });
      
      this.showRoundSummary = true;
      this.showResults = false;
      this.currentVideoData = null;
      
      // Démarrer la barre de progression
      this.roundTransitionProgress = 0;
      clearInterval(this.roundTransitionInterval);
      
      this.roundTransitionInterval = setInterval(() => {
        this.roundTransitionProgress += 1;
        if (this.roundTransitionProgress >= 100) {
          clearInterval(this.roundTransitionInterval);
        }
      }, 100); // 10 secondes au total (100 * 100ms)
    },
    
    handleNewRound(data) {
      this.currentRound = data.roundNumber;
      this.currentVideo = 1;
      this.showRoundSummary = false;
      this.skipsUsedThisRound = 0;
      clearInterval(this.roundTransitionInterval);
      console.log('New round started:', data);
    },

    handlePointsDeducted(data) {
      if (data.reason === 'skip_reveal') {
        alert(`-${data.points} points : Vous avez choisi de masquer votre identité`);
      }
    },

    // Utilitaires
    resetGameState() {
      this.currentGame = null;
      this.gameStatus = 'waiting';
      this.gamePin = '';
      this.players = [];
      this.isReady = false;
      this.currentRound = 1;
      this.currentVideo = 1;
      this.currentVideoData = null;
      this.hasVoted = false;
      this.showResults = false;
      this.videoResults = null;
      this.isVideoOwner = false;
      this.showRoundSummary = false;
      this.roundSummary = null;
      this.roundTransitionProgress = 0;
      clearInterval(this.roundTransitionInterval);
    },

    getPlayerNickname(playerId) {
      const player = this.players.find(p => p.id === playerId);
      return player ? player.nickname : 'Joueur Inconnu';
    },

    formatPlayerName(nickname) {
      if (!nickname) return 'Joueur Inconnu';
      // Si c'est un displayName, on le retourne tel quel
      if (!nickname.startsWith('@')) return nickname;
      // Si c'est un @ username, on enlève le @
      return nickname.substring(1);
    },

    formatNumber(num) {
      return new Intl.NumberFormat().format(num);
    },

    getTotalCorrectVotes() {
      if (!this.roundSummary || !this.roundSummary.playerCorrectVotes) return 0;
      return Object.values(this.roundSummary.playerCorrectVotes).reduce((total, votes) => total + votes, 0);
    },

    getTotalVotes() {
      if (!this.roundSummary || !this.roundSummary.playerTotalVotes) return 0;
      return Object.values(this.roundSummary.playerTotalVotes).reduce((total, votes) => total + votes, 0);
    },

    getTotalPoints() {
      if (!this.roundSummary || !this.roundSummary.playerScores) return 0;
      return Object.values(this.roundSummary.playerScores).reduce((total, score) => total + score, 0);
    },

    getAverageAccuracy() {
      if (!this.roundSummary || !this.roundSummary.playerAccuracy || !this.players.length) return 0;
      const accuracies = Object.values(this.roundSummary.playerAccuracy);
      if (accuracies.length === 0) return 0;
      return accuracies.reduce((total, accuracy) => total + accuracy, 0) / accuracies.length;
    },

    getPlayerScore(playerId) {
      if (!this.roundSummary || !this.roundSummary.playerScores) return 0;
      return this.roundSummary.playerScores[playerId] || 0;
    },

    getPlayerCorrectVotes(playerId) {
      if (!this.roundSummary || !this.roundSummary.playerCorrectVotes) return 0;
      return this.roundSummary.playerCorrectVotes[playerId] || 0;
    },

    getPlayerTotalVotes(playerId) {
      if (!this.roundSummary || !this.roundSummary.playerTotalVotes) return 0;
      return this.roundSummary.playerTotalVotes[playerId] || 0;
    },

    getPlayerAccuracy(playerId) {
      if (!this.roundSummary || !this.roundSummary.playerAccuracy) return 0;
      return this.roundSummary.playerAccuracy[playerId] || 0;
    },

    skipReveal() {
      if (!this.isVideoOwner || this.hasSkippedReveal || this.skipsUsedThisRound > 0) return;
      
      this.socket.emit('skipReveal', { cost: 5 });
      this.hasSkippedReveal = true;
      this.skipsUsedThisRound++;
    },

    startGame() {
      this.socket.emit('startGame');
    }
  }
};
</script>

<style>
.app {
  max-width: 1000px;
  margin: 0 auto;
  padding: 20px;
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  background: #f8f9fa;
  min-height: 100vh;
}

.header {
  text-align: center;
  margin-bottom: 30px;
  color: #2c3e50;
}

/* Cards */
.login-card, .menu-card, .waiting-card, .results-card, .end-card {
  background: white;
  border-radius: 15px;
  padding: 25px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

/* Login Screen */
.login-screen {
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 60vh;
}

.login-card {
  width: 100%;
  max-width: 400px;
}

/* Menu */
.menu-card {
  text-align: center;
}

.separator {
  margin: 20px 0;
  color: #666;
  position: relative;
}

.separator::before, .separator::after {
  content: '';
  position: absolute;
  top: 50%;
  width: 45%;
  height: 1px;
  background: #ddd;
}

.separator::before { left: 0; }
.separator::after { right: 0; }

/* Waiting Room */
.waiting-card {
  text-align: center;
}

.game-info {
  display: flex;
  justify-content: center;
  gap: 30px;
  margin: 20px 0;
}

.pin-display, .players-count {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.label {
  font-size: 0.9em;
  color: #666;
  margin-bottom: 5px;
}

.value {
  font-size: 1.2em;
  font-weight: bold;
  color: #2c3e50;
}

/* Game Play */
.game-play {
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.game-header {
  background: white;
  border-radius: 15px;
  padding: 15px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.progress-info {
  display: flex;
  gap: 20px;
}

.scores-panel {
  display: flex;
  gap: 20px;
}

.player-score {
  text-align: center;
  padding: 10px 20px;
  background: #f8f9fa;
  border-radius: 10px;
}

.score-header {
  margin-bottom: 5px;
}

.score-value {
  font-weight: bold;
  color: #2c3e50;
}

/* Video Display */
.game-area {
  background: white;
  border-radius: 15px;
  padding: 20px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  position: relative;
  min-height: 600px; /* Hauteur minimale pour éviter les sauts */
}

.video-container {
  display: flex;
  gap: 20px;
  margin-bottom: 20px;
}

.video-cover {
  width: 300px;
  height: 400px;
  object-fit: cover;
  border-radius: 10px;
}

.video-stats {
  flex-grow: 1;
  display: flex;
  flex-direction: column;
  justify-content: center;
  gap: 15px;
}

.stat {
  display: flex;
  align-items: center;
  gap: 10px;
  font-size: 1.1em;
}

.icon {
  font-size: 1.5em;
}

/* Voting Section */
.voting-section {
  margin-top: 20px;
  text-align: center;
}

.votes-counter {
  margin: 10px 0;
  font-size: 1.1em;
  color: #666;
  text-align: center;
}

.votes-progress-bar {
  width: 200px;
  height: 8px;
  background: #eee;
  border-radius: 4px;
  margin: 8px auto;
  overflow: hidden;
}

.votes-progress {
  height: 100%;
  background: #28a745;
  border-radius: 4px;
  transition: width 0.3s ease;
}

.vote-buttons {
  display: flex;
  justify-content: center;
  gap: 15px;
  margin-top: 15px;
}

/* Results */
.results-card {
  padding: 30px;
}

.video-owner {
  text-align: center;
  margin-bottom: 20px;
  padding-bottom: 20px;
  border-bottom: 1px solid #eee;
}

.votes-summary {
  display: flex;
  flex-direction: column;
  gap: 15px;
}

.vote {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px;
  background: #f8f9fa;
  border-radius: 8px;
}

.vote-player {
  display: flex;
  align-items: center;
  gap: 10px;
}

.vote-result {
  font-size: 1.2em;
}

.vote-points {
  display: flex;
  align-items: baseline;
  gap: 5px;
}

.points {
  font-size: 1.2em;
  font-weight: bold;
  color: #2c3e50;
}

.unit {
  color: #666;
  font-size: 0.9em;
}

/* Game End */
.end-card {
  text-align: center;
}

.final-scores {
  margin: 30px 0;
}

.final-score {
  display: flex;
  align-items: center;
  gap: 20px;
  padding: 15px;
  margin: 10px 0;
  background: #f8f9fa;
  border-radius: 10px;
}

.rank {
  width: 30px;
  height: 30px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: #2c3e50;
  color: white;
  border-radius: 50%;
  font-weight: bold;
}

/* Common Elements */
button {
  padding: 12px 24px;
  border: none;
  border-radius: 8px;
  background: #007bff;
  color: white;
  cursor: pointer;
  font-size: 1em;
  transition: background 0.3s;
}

button:hover {
  background: #0056b3;
}

button:disabled {
  background: #ccc;
  cursor: not-allowed;
}

.primary-btn {
  background: #28a745;
}

.primary-btn:hover {
  background: #218838;
}

input {
  padding: 12px;
  border: 1px solid #ddd;
  border-radius: 8px;
  font-size: 1em;
  width: 100%;
  margin-bottom: 10px;
}

.streak {
  color: #ff4500;
  font-weight: bold;
}

.correct {
  color: #28a745;
}

.incorrect {
  color: #dc3545;
}

.voted {
  opacity: 0.7;
}

h2 {
  color: #2c3e50;
  margin-bottom: 20px;
}

h3 {
  color: #2c3e50;
  margin-bottom: 15px;
}

.player-name {
  font-weight: 500;
  color: #2c3e50;
}

/* Styles pour l'écran du propriétaire de la vidéo */
.video-owner-screen {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: space-between;
  background-color: #121212;
  color: white;
  border-radius: 15px;
  overflow: hidden;
  height: 600px;
  padding: 20px;
}

.video-frame {
  position: relative;
  width: 300px;
  height: 300px;
  border-radius: 20px;
  overflow: hidden;
  margin-bottom: 20px;
}

.video-cover-owner {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.emoji-overlay {
  position: absolute;
  bottom: 0;
  left: 0;
  width: 100%;
  height: 40%;
  background: linear-gradient(transparent, rgba(0,0,0,0.7));
  display: flex;
  justify-content: center;
  align-items: flex-end;
  padding-bottom: 10px;
}

.emoji {
  font-size: 60px;
  animation: float 2s ease-in-out infinite;
}

@keyframes float {
  0%, 100% { transform: translateY(0); }
  50% { transform: translateY(-10px); }
}

.owner-message {
  text-align: center;
  margin: 20px 0;
}

.owner-title {
  font-family: Impact, Haettenschweiler, 'Arial Narrow Bold', sans-serif;
  font-size: 32px;
  text-transform: uppercase;
  margin: 5px 0;
  letter-spacing: 1px;
  text-shadow: 2px 2px 4px rgba(0,0,0,0.5);
}

.owner-subtitle {
  font-size: 18px;
  margin-top: 15px;
  opacity: 0.9;
}

.owner-actions {
  width: 100%;
  margin-top: 20px;
}

.secret-text {
  text-align: center;
  margin-bottom: 15px;
  font-size: 18px;
}

.action-bar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  background-color: #333;
  border-radius: 30px;
  padding: 10px 20px;
}

.skip-button {
  background-color: #ff4500;
  color: white;
  border: none;
  padding: 10px 20px;
  border-radius: 20px;
  cursor: pointer;
  transition: all 0.3s ease;
  opacity: 1;
}

.skip-button:hover:not(:disabled) {
  background-color: #ff6b6b;
  transform: translateY(-2px);
}

.skip-button:disabled {
  background-color: #ccc;
  cursor: not-allowed;
  opacity: 0.7;
}

.counter {
  background-color: #444;
  padding: 5px 10px;
  border-radius: 15px;
  font-size: 14px;
}

/* Styles pour le résumé du round */
.round-summary {
  display: flex;
  justify-content: center;
  align-items: center;
}

.summary-card {
  background: white;
  border-radius: 15px;
  padding: 30px;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1);
  width: 100%;
  max-width: 800px;
  text-align: center;
}

.round-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 20px;
}

.round-badge {
  background: #28a745;
  padding: 5px 10px;
  border-radius: 5px;
  color: white;
  font-weight: bold;
}

.round-stats {
  display: flex;
  justify-content: space-between;
  margin-bottom: 20px;
}

.stat-box {
  text-align: center;
  flex: 1;
}

.stat-icon {
  font-size: 24px;
  margin-bottom: 5px;
}

.stat-value {
  font-size: 18px;
  font-weight: bold;
  color: #2c3e50;
}

.stat-label {
  font-size: 12px;
  color: #666;
}

.best-player {
  display: flex;
  align-items: center;
  justify-content: center;
  margin: 30px 0;
  padding: 20px;
  background: linear-gradient(to right, #f8f9fa, #e9ecef);
  border-radius: 15px;
}

.trophy {
  font-size: 50px;
  margin-right: 20px;
}

.best-player-info {
  text-align: left;
}

.best-player-title {
  font-size: 14px;
  color: #666;
  margin-bottom: 5px;
}

.best-player-name {
  font-size: 24px;
  font-weight: bold;
  color: #2c3e50;
  margin-bottom: 5px;
}

.best-player-score {
  font-size: 18px;
  color: #28a745;
  font-weight: bold;
}

.player-stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
  gap: 20px;
  margin: 30px 0;
}

.player-round-stats {
  background: white;
  padding: 20px;
  border-radius: 15px;
  box-shadow: 0 4px 6px rgba(0,0,0,0.1);
  transition: transform 0.2s;
}

.player-round-stats:hover {
  transform: translateY(-2px);
}

.player-round-header {
  margin-bottom: 15px;
  padding-bottom: 10px;
  border-bottom: 1px solid #eee;
}

.stats-row {
  display: flex;
  justify-content: space-between;
}

.player-progress {
  margin-top: 15px;
}

.progress-bar-label {
  font-size: 12px;
  color: #666;
  margin-bottom: 5px;
}

.progress-bar {
  height: 10px;
  background: #eee;
  border-radius: 5px;
  margin-top: 5px;
  overflow: hidden;
}

.progress {
  height: 100%;
  background: #28a745;
  border-radius: 5px;
  transition: width 0.1s linear;
}

.high-accuracy {
  background: #ff4500;
}

.winner-badge {
  background: linear-gradient(45deg, #ff4500, #ff6b6b);
  color: white;
  padding: 4px 10px;
  border-radius: 12px;
  font-size: 0.9em;
  font-weight: bold;
}

.next-round-info {
  margin-top: 30px;
}

.progress-bar {
  height: 10px;
  background: #eee;
  border-radius: 5px;
  margin-top: 15px;
  overflow: hidden;
}

.progress {
  height: 100%;
  background: #28a745;
  border-radius: 5px;
  transition: width 0.1s linear;
}

.player-badges {
  display: flex;
  gap: 8px;
  margin-top: 4px;
}

.combo-badge {
  background: linear-gradient(45deg, #ff4d4d, #ff9933);
  color: white;
  padding: 2px 8px;
  border-radius: 12px;
  font-size: 0.9em;
  animation: pulse 1s infinite;
}

.bonus-badge {
  background: linear-gradient(45deg, #ffd700, #ffa500);
  color: white;
  padding: 2px 8px;
  border-radius: 12px;
  font-size: 0.9em;
}

.points-breakdown {
  display: flex;
  gap: 4px;
  align-items: center;
  font-size: 0.9em;
}

.base-points {
  color: #28a745;
}

.combo-points {
  color: #ff4500;
  animation: pulse 1s infinite;
}

@keyframes pulse {
  0% { opacity: 1; }
  50% { opacity: 0.7; }
  100% { opacity: 1; }
}

.vote-details {
  display: flex;
  align-items: center;
  gap: 8px;
}

.player-score {
  background: white;
  padding: 12px 16px;
  border-radius: 12px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
  transition: transform 0.2s;
}

.player-score:hover {
  transform: translateY(-2px);
}

.score-header {
  display: flex;
  flex-direction: column;
}

.score-value {
  font-size: 1.4em;
  font-weight: bold;
  color: #2c3e50;
  margin-top: 4px;
}

.vote {
  background: white;
  padding: 12px 16px;
  border-radius: 12px;
  margin-bottom: 8px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
  transition: transform 0.2s;
}

.vote:hover {
  transform: translateY(-2px);
}

.vote-player {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.vote-points {
  display: flex;
  align-items: baseline;
  gap: 4px;
}

.points {
  font-size: 1.4em;
  font-weight: bold;
  color: #2c3e50;
}

.unit {
  color: #666;
  font-size: 0.9em;
}

.reveal-screen {
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 400px;
}

.reveal-card {
  background: white;
  border-radius: 15px;
  padding: 30px;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1);
  width: 100%;
  max-width: 600px;
  text-align: center;
}

.reveal-header {
  margin-bottom: 30px;
}

.reveal-header h2 {
  font-size: 2em;
  color: #2c3e50;
}

.reveal-content {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 30px;
  opacity: 0;
  animation: fadeIn 0.5s ease forwards;
}

.video-owner-reveal {
  display: flex;
  align-items: center;
  gap: 30px;
  padding: 20px;
  background: linear-gradient(to right, #f8f9fa, #e9ecef);
  border-radius: 15px;
  width: 100%;
}

.video-cover-reveal {
  width: 150px;
  height: 200px;
  object-fit: cover;
  border-radius: 10px;
}

.owner-info {
  text-align: left;
}

.reveal-text {
  font-size: 1.2em;
  color: #666;
  margin-bottom: 10px;
}

.owner-name {
  font-size: 2em;
  color: #2c3e50;
  font-weight: bold;
  opacity: 0;
  animation: revealName 0.8s ease forwards;
  animation-delay: 0.5s;
}

.reveal-button {
  background: linear-gradient(45deg, #ff4500, #ff6b6b);
  color: white;
  padding: 15px 30px;
  border-radius: 30px;
  font-size: 1.2em;
  border: none;
  cursor: pointer;
  transition: transform 0.2s, box-shadow 0.2s;
}

.reveal-button:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 15px rgba(255, 69, 0, 0.3);
}

/* Transitions améliorées */
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.5s ease, transform 0.5s ease;
  position: absolute;
  width: 100%;
}

.fade-enter-from {
  opacity: 0;
  transform: translateX(30px);
}

.fade-leave-to {
  opacity: 0;
  transform: translateX(-30px);
}

.fade-enter-to,
.fade-leave-from {
  opacity: 1;
  transform: translateX(0);
}

.game-area {
  position: relative;
  min-height: 600px; /* Hauteur minimale pour éviter les sauts */
}

.video-display,
.reveal-screen,
.results {
  width: 100%;
}

/* Animation de la révélation */
.reveal-content {
  opacity: 0;
  animation: fadeIn 0.5s ease forwards;
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.owner-name {
  opacity: 0;
  animation: revealName 0.8s ease forwards;
  animation-delay: 0.5s;
}

@keyframes revealName {
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.blurred {
  filter: blur(8px);
  pointer-events: none;
}

.hidden {
  display: none;
}

.skipped-text {
  font-size: 1.2em;
  color: #666;
  font-style: italic;
  text-align: center;
  margin-top: 20px;
}

.anonymous {
  font-style: italic;
  color: #999;
}

/* Écran de chargement */
.loading-screen {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.7);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

.loading-spinner {
  text-align: center;
  color: white;
}

.spinner {
  width: 50px;
  height: 50px;
  border: 5px solid #f3f3f3;
  border-top: 5px solid #3498db;
  border-radius: 50%;
  animation: spin 1s linear infinite;
  margin: 0 auto 20px;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

/* Ajouter le style pour le texte de limitation */
.limit-text {
  font-size: 0.8em;
  opacity: 0.8;
  display: block;
  margin-top: 4px;
}

.start-btn {
  background: #ff4500;
  color: white;
  padding: 12px 24px;
  border-radius: 8px;
  font-size: 1.1em;
  border: none;
  cursor: pointer;
  transition: all 0.3s ease;
}

.start-btn:hover {
  background: #ff6b6b;
  transform: translateY(-2px);
}
</style>