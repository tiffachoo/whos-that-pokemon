<template>
	<div id="app">
		<main 
			:class="{'poke-classic': classic}"
			class="container"
		>
			<transition name="animate-section">
				<section 
					v-if="!isPlaying && !isDone" 
					class="poke-section"
				>
					<h1 
						aria-live="polite"
						class="h2"
					>What type of trainer are you?</h1>
					<div class="poke-intro-trainer">
						<div class="poke-ball"></div>
						<img 
							:class="{ active: trainerHovered === 'classic' }"
							class="poke-trainer-img poke-trainer-img-classic"
							src="@/assets/trainers/red-rb.png" 
							alt="Trainer red from red and blue"
						>
						<img 
							:class="{ active: trainerHovered === 'master' }"
							class="poke-trainer-img poke-trainer-img-master"
							src="@/assets/trainers/red-sm.png" 
							alt="Trainer red from sun and moon"
						>
					</div>
					<button 
						class="button spacer"
						@click="startGame(151)"
						@mouseover="trainerHover('classic')"
						@mouseout="trainerHover"
					>
						Classic
					</button>
					<button 
						class="button"
						@click="startGame(0)"
						@mouseover="trainerHover('master')"
						@mouseout="trainerHover"
					>
						Master
					</button>
				</section>
			</transition>
			
			<transition name="animate-section">
				<section
					v-if="isPlaying"
					class="poke-section"
				>
					<h1 
						aria-live="polite"
						class="poke-title"
					>
						Who's that pokemon?
					</h1>
					<div class="poke-question-wrapper">
						<h2 class="poke-question">
							<span 
								aria-live="polite"
								class="poke-question-number"
							>
								<span class="sr-only">Question:</span>
								{{ question }}
							</span>
							<span class="poke-question-amount">
								/ {{ questionAmount}}
							</span>
						</h2>
						<span class="poke-score">
							{{ score }}
							<small>pts</small>
						</span>
						<div 
							class="poke-image"
							:class="{ 'poke-image-success': isChecked && selected.name === answer.name, 'poke-image-error': isChecked && selected.name !== answer.name }"
						>
							<img 
								:src="image" 
								alt="No cheating"
								class="poke-image-img"
							>
						</div>
						<transition-group 
							tag="div"
							name="animate-options"
							:class="{'poke-options-answers': isChecked}"
							class="poke-options"
						>
							<div 
								v-for="(pokemon, index) in options"
								:key="pokemon.name"
								:data-index="index"
								:class="{ 'selected': selected.index === index, 'success': isChecked && pokemon.name === answer.name, 'error': isChecked && selected.index === index && selected.name !== answer.name }"
								class="poke-options-button"
								@click="selectAnswer(pokemon.name, index)"
							>
								<input 
									:id="pokemon.name"
									:value="pokemon.name"
									class="poke-options-control"
									name="poke"
									type="radio"
								>
								<label :for="pokemon.name">
									{{ pokemon.name | prettifyName }}
								</label>
							</div>
						</transition-group>
						<footer class="poke-buttons">
							<button
								:disabled="isChecked || Object.keys(selected).length < 1"
								class="button"
								@click="checkAnswer"
							>Submit</button>
							<span 
								v-if="isChecked" class="sr-only"
								aria-live="polite"
							>
								<span v-if="selected.name === answer.name">
									Correct!
								</span>
								<span v-else>
									Incorrect, the correct answer was {{ answer.name }}.
								</span>
							</span>
							<button 
								:disabled="!isChecked"
								class="button"
								@click="getNextQuestion"
							>Next</button>
						</footer>
					</div>
				</section>
			</transition>
			
			<transition name="animate-section">
				<section
					v-if="isDone"
					class="poke-final"
				>
					<h1 
						aria-live="polite"
						class="h2"
					>Final score</h1>
					<span class="poke-final-score">
						<span class="poke-final-score-number">{{ score }}</span>
						pts
					</span>
					<button 
						class="button"
						@click="restartGame"
					>Play again</button>
				</section>
			</transition>
		</main>
	</div>
</template>

<script>
const pkmnTotal = 802;
const url = `https://pokeapi.co/api/v2/pokemon/?limit=${pkmnTotal}`;
const optionAmount = 4;
let pokemonData = [];
const prettyNames = {
	'nidoran-f': 'nidoran♀',
	'nidoran-m': 'nidoran♂',
	'mr-mime': 'mr. mime',
	'deoxys-normal': 'deoxys',
	'wormadam-plant': 'wormadam',
	'mime-jr': 'mime jr.',
	'giratina-altered': 'giratina',
	'shaymin-land': 'shaymin',
	'basculin-red-striped': 'basculin',
	'darmanitan-standard': 'darmanitan',
	'tornadus-incarnate': 'tornadus',
	'thundurus-incarnate': 'thundurus',
	'landorus-incarnate': 'landorus',
	'keldeo-ordinary': 'keldeo',
	'meloetta-aria': 'meloetta',
	'meowstic-male': 'meowstic',
	'aegislash-shield': 'aegislash',
	'pumpkaboo-average': 'pumpkaboo',
	'gourgeist-average': 'gourgeist',
	'oricorio-baile': 'oricorio',
	'lycanroc-midday': 'lycanroc',
	'wishiwashi-solo': 'wishiwashi',
	'type-null': 'type: null',
	'minior-red-meteor': 'minior',
	'mimikyu-disguised': 'mimikyu',
	'tapu-koko': 'tapu koko',
	'tapu-lele': 'tapu lele',
	'tapu-bulu': 'tapu bulu',
	'tapu-fini': 'tapu fini'
};

export default {
	name: 'App',
	filters: {
		prettifyName(name) {
			return prettyNames[name] || name;
		}
	},
	data() {
		return {
			pokemon: [],
			pkmnAmount: null,
			score: 0,
			question: 0,
			questionAmount: 10,
			answer: {},
			selected: {},
			options: [],
			isPlaying: false,
			isDone: false,
			isChecked: false,
			trainerHovered: null
		}
	},
	computed: {
		image() {
			const url = './assets/pokemon/';
			const imageUrl = `${url}${this.classic ? 'redblue' : 'sunmoon'}/`
			const number = this.answer.url.match(/\/(\d+)/)[1];
			return require(`${imageUrl}${number}.png`);
		},
		classic() {
			return this.pkmnAmount <= 151;
		}
	},
	mounted() {
		const pokeList = localStorage.getItem('pokeList');
		
		if (pokeList) {
			pokemonData = JSON.parse(pokeList);
		} else {
			this.getData()
				.then(res => {
					pokemonData = res.results;
					localStorage.setItem('pokeList', JSON.stringify(res.results));
			});
		}
	},
	methods: {
		getData() {
			return fetch(url)
				.then(res => res.json())
				.catch(err => console.log(err));
		},
		startGame(val) {
			this.question = 0;
			this.score = 0;
			this.isPlaying = true;
			this.pokemon = [...pokemonData];
			
			this.pkmnAmount = val || pkmnTotal;
			
			this.getNextQuestion();
		},
		getNextQuestion() {
			document.body.focus();
			this.question += 1;
			this.resetAnswer();
			
			if (this.question <= this.questionAmount) {
				let removed = '';
				for (let i = 1; i <= optionAmount; i++) {
					removed = this.pokemon.splice(this.getRandomPokemon(i), 1)[0];
					if (i === 1) {
						this.answer = removed;
					} else {
						this.options.push(removed);
					}
				}

				const pos = Math.floor(Math.random() * optionAmount);
				this.options.splice(pos, 0, this.answer);
			} else {
				this.isPlaying = false;
				this.isDone = true;
				this.resetAnswer();
			}
		},
		selectAnswer(ans, index) {
			if (!this.isChecked) {
				this.$set(this.selected, 'name', ans);
				this.$set(this.selected, 'index', index);
			}
		},
		checkAnswer() {
			this.isChecked = true;
			
			if (this.selected.name === this.answer.name) {
				this.score += 10;
			}
		},
		getRandomPokemon(index) {
			const diff = (this.question - 1) * 4 + index;
			return Math.floor(Math.random() * (this.pkmnAmount + 1 - diff));
		},
		resetAnswer() {
			this.options = [];
			this.selected = {};
			this.answer = {};
			this.isChecked = false;
		},
		restartGame() {
			this.isDone = false;
		},
		trainerHover(val) {
			this.trainerHovered = val;
		}
	}
}
</script>

<style lang="scss">
@import '@/styles/reset';
@import '@/styles/styles';
</style>
