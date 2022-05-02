<template>
	<header>
		<h1 data-shadow="Multi Blind Test">Multi Blind Test</h1>
		<h3>Maker</h3>
	</header>

	<div class="players_wrapper">
		<div class="player-item" v-for="(player, idx) in players" :key="player.ID">
			<div class="volume_slider">
				<input type="range" v-model="player.volume" class="custom-range" min="0" max="100" step="5" @input="setVolume(player.ID)" @change="setVolume(player.ID)">
			</div>

			<button class="btn cst_btn" @click="mute(player.ID)" :class="{active: player.muted}">
				<i v-show="!player.muted" class="fas fa-volume-mute"></i>
				<i v-show="player.muted" class="fas fa-volume-up"></i>
			</button>

			<div v-if="!isMystery" class="name_video flex-fill">
				<a :href="'https://www.youtube.com/watch?v='+player.youtubeID" target="_blank">{{ player.title }}</a>
				<span class="author">- {{ player.author }}</span>
			</div>

			<div v-else class="name_video flex-fill">
				<div class="mystery">Musique mystère n° <strong>{{ idx }}</strong></div>
			</div>

			<div class="offset_input">
				<i class="fas fa-stopwatch"></i>
				<div>
					<input title="Décalage au début" type="number" v-model="player.offset" min="0" step=".5"> s
				</div>
			</div>
			
			<button class="btn cst_btn danger" @click="removePlayer(player.ID)">
				<i class="fas fa-times"></i>
			</button>

			<YouTube class="youtube_player"
				:src="player.youtubeID"
				:ref="'player_'+player.ID" 
				:vars="playerVars"
				@ready="setQuality(player.ID)"
				width="1" height="1"
				:host="host"
			/>
		</div>
		<button id="add_video_zone" @click="showMediaAdding = true"><i class="fas fa-plus"></i> Ajouter un titre via Toutube</button>
	</div>

	<div id="mystery_panel">
		<label class="switch">
			<input type="checkbox" v-model="isMystery">
			<span class="slider"></span>
		</label>
		<div class="label">Mode mystère</div>
	</div>

	<div id="controller">
		<div class="copyright">Made by <strong>Blyy</strong></div>
		<div class="shortcuts">
			<h4>Raccourcis clavier</h4>
			<p><strong>[ ESPACE ]</strong> Play / Pause</p>
			<p><strong>[ &larr; ]</strong> Lire depuis le début</p>
			<p><strong>[ + ]</strong> Ajouter une piste Youtube</p>
		</div>

		<div class="commandes" v-show="Object.keys(players).length">

			<button class="btn cst_btn md mx-2" @click="muteAllPlayers">
				<i v-show="!allMuted" class="fas fa-volume-mute"></i>
				<i v-show="allMuted" class="fas fa-volume-up"></i>
			</button>

			<button class="btn cst_btn lg mx-2" @click="playPauseBtn">
				<i v-show="!isPlayed" class="fas fa-play"></i>
				<i v-show="isPlayed" class="fas fa-pause"></i>
			</button>

			<button id="reset_btn" class="btn cst_btn md mx-2" @click="resetAll">
				<i class="fas fa-step-backward"></i>
			</button>
			<!-- <i class="fas fa-undo-alt"></i> -->
		</div>
			
		<div class="exports_cmds">
			<a @click.prevent="openPanel(1)">Importer</a>
			<a @click.prevent="openPanel(2)">Exporter</a>
			<a @click.prevent="openPanel(3)"><i class="fas fa-info-circle"></i> Règles</a>
		</div>
	</div>

	<transition name="fade">
		<div class="add_media_panel" v-show="showMediaAdding" ref="panel_media">
			<div class="close_btn" @click="showMediaAdding = false">×</div>
			<div class="media_inner">
				<div class="left">
					<div class="bloc search">
						<h3>Chercher une vidéo sur youtube</h3>
						<form v-on:submit.prevent="youtubeSearch">
							<input v-model="yt_search" type="text" placeholder="Rechercher sur youtube">
							<button @click="youtubeSearch"><i class="fas fa-search"></i></button>
						</form>
					</div>
					<div class="bloc with_id">
						<h3>Ajouter via l'Id ou l'Url Youtube</h3>
						<form v-on:submit.prevent="addMedia">
							<input type="text" v-model="yt_import_id" placeholder="ex: dQw4w9WgXcQ">
							<button @click="addMedia">Ajouter</button>
						</form>
					</div>
					<div class="bloc random">
						<h3>Au hasard</h3>
						<button @click="addRandom">Ajouter une musique au pif !</button>					
					</div>
				</div>
				<div class="right">					
					<div class="youtube_results">
						<div v-for="result in youtubeResults" class="result_item" @click="addYoutubeSearch(result)">
							<div class="thumbnail"><img :src="result.snippet.thumbnails.default.url" alt="" /></div>
							<div class="content">								
								<div class="title">{{ result.snippet.title }}</div>
								<div class="author">{{ result.snippet.channelTitle }}</div>
							</div>
						</div>
					</div>
				</div>
			</div>
		</div>
	</transition>

	<transition name="fade">
		<div class="panel_content" v-show="showPanel" ref="panel_content">
			<div class="close_btn" @click="showPanel = false">×</div>
			
			<div class="content_inner" v-show="contentPanel === 1">
				<h2>Importer vos paramètres</h2>
				<textarea v-model="jsonImport" ref="import_zone" placeholder="Coller les paramètres en JSON ici"></textarea>
				<button @click="importJson">Importer</button>
			</div>

			<div class="content_inner" v-show="contentPanel === 2">
				<h2>Exporter les paramètres</h2>
				<pre ref="export_json">{{ exportParams }}</pre>
				<button @click="copyJson"><i class="far fa-clipboard"></i> Copier dans le presse-papiers</button>
			</div>

			<div class="content_inner" v-show="contentPanel === 3">
				<h2>Règle du Multi Blind Test</h2>
				<ul>
					<li>Tous les morceaux à trouver sont lu en même temps</li>
					<li>Il faut trouver les différents titres par leur nom ou leur auteur</li>
					<li>Dés qu'un titre est trouvé, il est retiré</li>
					<li>Les deux derniers titres doivent êtres trouvés en même temps</li>
				</ul>
			</div>
		</div>
	</transition>
	<div class="overlay" v-show="showPanel" @click="showPanel = false"></div>

	<YouTube id="blank_video" class="youtube_player" src="t1-SdYWIBB4" vars="playerVars" width="1" height="1" :host="host" />
</template>

<script>
	import YouTube from 'vue3-youtube'

	export default {
		data() {
			return {
				host : "https://blyydev.github.io/multiblindtest-maker/dist/",
				yt_search : '',
				yt_import_id : '',
				isPlayed: false,
				isMystery: false,
				allMuted: false,
				players : {},
				playerVars: {
					mute: 0,
					autoplay: 0,
					showinfo: 0,
					modestbranding: 1,
					controls: 0,
					disablekb: 1,
					enablejsapi: 1
				},
				jsonImport: '',
				showPanel: false,
				showMediaAdding: false,
				youtubeResults : [],
				contentPanel : 0,
				testId: [
					'Bq70BFHYfA8',
					'ouq343rCORw',
					'RK3J2sDEQ1M',
					'V1thzqeJjqs',
					'hrB-_nIer88',
					'pIgZ7gMze7A',
					'djV11Xbc914',
					'dQw4w9WgXcQ',
					'sOnqjkJTMaA',
					'79fzeNUqQbQ',
					'mwgZalAFNhM',
					'otCpCn0l4Wo',
					'KQ6zr6kCPj8',
					'ReEgXh-wURs',
					'nfWlot6h_JM',
					'4m1EFMoRFvY',
					'6M6samPEMpM',
					'4NJH75q0Syk',
					'UG3VcCAlUgE',
					'y8OtzJtp-EM',
					'NCtzkaL2t_Y',
					'hTWKbfoikeg',
					'lWD9wVPwhH4',
					'kXYiU_JCYtU',
					'zUwEIt9ez7M',
					'7iNbnineUCI',
					'hOmE6-LU4Hk',
					'N3oCS85HvpY',
					'CD-E-LDc384',
					'ye5BuYf8q4o',
				]
			}
		},
		methods: {
			addMedia(){
				let tmpYoutubeLink = this.yt_import_id
				let nextId = Object.keys(this.players).length + 1;
				let ytID = null

				if(tmpYoutubeLink.includes('youtube.com')){
					ytID = this.getIdfromUrl(tmpYoutubeLink)
				}else{
					ytID = tmpYoutubeLink
				}

				if(ytID != null){
					this.resetAll(false)
					let playerTMP = {
						ID: nextId,
						youtubeID: ytID,
						title: null,
						author: null,
						volume: 80,
						muted: false,
						offset: 0
					}
					this.players[nextId] = playerTMP
					this.getTitle(nextId)
				}

				this.yt_import_id = ""
				this.showMediaAdding = false
			},
			addRandom(){
				let randomIndex = Math.floor(Math.random() * this.testId.length);
				let randomElement = this.testId[randomIndex];
					this.testId.splice(randomIndex, 1);
				let nextId = Object.keys(this.players).length + 1;

				if(randomElement != null){
					this.resetAll(false)
					let playerTMP = {
						ID: nextId,
						youtubeID: randomElement,
						title: null,
						author: null,
						volume: 80,
						muted: false,
						offset: 0
					}

					this.players[nextId] = playerTMP
					this.getTitle(nextId)
				}
				this.showMediaAdding = false
			},
			addYoutubeSearch(video){
				let nextId = Object.keys(this.players).length + 1;
				this.resetAll(false)

				let playerTMP = {
					ID: nextId,
					youtubeID: video.id.videoId,
					title: video.snippet.title,
					author: video.snippet.channelTitle,
					volume: 80,
					muted: false,
					offset: 0
				}
				this.players[nextId] = playerTMP

				// Reset panel Search
				this.yt_search = ""
				this.showMediaAdding = false
			},
			getTitle(id){
				let urlApi = "https://noembed.com/embed?url=https://www.youtube.com/watch?v=";
				fetch(urlApi+this.players[id].youtubeID)
					.then(response => response.json())
					.then(data => {
						this.players[id].title = data.title
						this.players[id].author = data.author_name
					});
			},
			playPauseBtn(){
				if(this.isPlayed){
					this.pauseAll()
				}else{
					this.playAll()
				}
			},
			playAll() {
				let _this = this
				for(let x in this.players) { 
					let tmpRef = 'player_'+x; 
					this.$refs[tmpRef].playVideo()
				}
				this.isPlayed = true
			},
			pauseAll(){
				let _this = this
				for(let x in this.players) { 
					let tmpRef = 'player_'+x; 
					this.$refs[tmpRef].pauseVideo()
				}
				this.isPlayed = false
			},
			muteAllPlayers(){
				for(let x in this.players){
					let tmpRef = 'player_'+x; 
					this.$refs[tmpRef].playVideo()

					if(this.allMuted){
						this.$refs[tmpRef].unMute()
						this.$refs[tmpRef].setVolume(this.players[x].volume)
						this.players[x].muted = false
					}else{
						this.$refs[tmpRef].setVolume(0)
						this.$refs[tmpRef].mute()
						this.players[x].muted = true
					}
				}
				this.allMuted = !this.allMuted
			},
			resetAll(playAfter = true){
				let _this = this
				this.isPlayed = false
				for(let x in this.players) { 
					let tmpRef = 'player_'+x;
					this.$refs[tmpRef].player.seekTo(this.players[x].offset)
					this.$refs[tmpRef].pauseVideo()
				}
				if(playAfter){
					this.playAll()
				}
			},
			setVolume(id){
				this.$refs['player_'+id].setVolume(this.players[id].volume)
				this.players[id].muted = (this.players[id].volume > 0) ? false : true
			},
			mute(id) {
				let player = this.$refs['player_'+id]

				if( this.players[id].muted ){
					player.unMute()
					player.setVolume(this.players[id].volume)
				}else{
					player.setVolume(0)
					player.mute()
				}
				this.players[id].muted = !this.players[id].muted
			},
			setQuality(id){
				let player = this.$refs['player_'+id]
				if(this.players[id].muted){
					player.setVolume(0)
				}else {					
					player.unMute()
					player.setVolume(this.players[id].volume)
				}
				player.setPlaybackQuality('small')
			},
			removePlayer(id){
				delete this.players[id]
			},
			getIdfromUrl(url){
				var regExp = /^.*((youtu.be\/)|(v\/)|(\/u\/\w\/)|(embed\/)|(watch\?))\??v?=?([^#&?]*).*/;
				var match = url.match(regExp);
				return (match&&match[7].length==11)? match[7] : false;
			},
			copyJson(){
				navigator.clipboard.writeText(JSON.stringify(this.players, null, 2));
				this.showPanel = false
			},
			importJson(){
				let newParams = ''
				try{
					newParams = JSON.parse(this.jsonImport);
				}catch (e) {
					return false;
				}

				if (typeof newParams === "object" && newParams !== null) {
					for(let y in newParams) { 
						console.log('>>', newParams[y].title)
						this.players[y] = newParams[y]
					}
					this.showPanel = false
					return true;
				}

				return false;
			},
			openPanel(id){
				this.showPanel = true
				this.contentPanel = id
				if(id == 1){
					setTimeout(() => { this.$refs['import_zone'].focus() }, 50)
				}
			},
			youtubeSearch(){
				let videoNumber = 20
				let apiKey = 'AIzaSyC84EvoAS6adFAbHgMiYwJLw3z8I5Hkpsg'

				let url = 'https://www.googleapis.com/youtube/v3/search?part=snippet&type=video&maxResults='+videoNumber+'&key='+apiKey+'&q='+this.yt_search

				fetch(url)
					.then(response => response.json())
					.then(data => {
						this.youtubeResults = data.items
					});
			}
		},
		computed: {	
			exportParams: function(){
				return JSON.stringify(this.players, null, 2);
			}
		},
		mounted() {
			window.addEventListener("keydown", e => {
				if(this.showMediaAdding) return false;

				// Play / pause
				if(e.keyCode === 32 && Object.keys(this.players).length){
					e.preventDefault()
					if(this.isPlayed){
						this.pauseAll()
					}else{
						this.playAll()
					}
				}
				// Reset
				else if(e.keyCode === 37 && Object.keys(this.players).length) this.resetAll()

				// Add video	
				else if(e.keyCode === 107) this.showMediaAdding = true
			});

			// Remvoe Blank Video - for Init YT api
			setTimeout(() => { document.getElementById('blank_video').remove() }, 250)
		},
		unmounted(){
			this.players = {}
		},
		components: { YouTube }
	}
</script>

<style lang="less">
	@import "assets/styles.less";
</style>