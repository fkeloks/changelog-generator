<template>
    <div class="card" style="margin: 60px 0 60px 0;">
        <div class="card-header">
            <h3>Githug API : Genération de changelog</h3>
        </div>
        <div class="card-body">
            <div class="card-title" v-show="owner && repository">
                Dépôt :
                <a :href="url" target="_blank">
                    {{ url }}<span v-if="!error && brancheState === null"> &#10004;</span>
                </a>
                (Branche : {{ branchSelected }})
            </div>
            <div class="card-text">
                <div class="form-group">
                    <input type="text" class="form-control" v-model="owner" @keyup.enter="searchCommits" placeholder="Utilisateur">
                </div>
                <div class="form-group">
                    <input type="text" class="form-control" v-model="repository" @keyup.enter="searchCommits" placeholder="Dépôt">
                </div>
                <div class="form-group" v-if="owner != null && repository != null">
                    <label for="branche">Branche</label>
                    <select class="form-control form-control-sm" id="branche" v-if="availabesBranches.length != 0" v-model="branchSelected">
                        <option v-for="branche in availabesBranches">{{ branche.name }}</option>
                    </select>
                    <select class="form-control form-control-sm" id="branche" v-if="brancheState != null && availabesBranches.length === 0">
                        <option selected>{{ brancheState }}</option>
                    </select>
                </div>
                <button type="submit" @click.prevent="searchCommits" class="btn btn-default">Afficher les commits
                </button>
                <button type="submit" @click.prevent="owner = 'fkeloks', repository = 'simple-container'" class="btn btn-default">
                    Fausses données
                </button>
                <button type="submit" @click.prevent="reset" class="btn btn-default">Réinitialiser</button>
                <button class="btn btn-default" @click.prevent="exportJSON">Exporter en JSON</button>
                <button class="btn btn-default" @click.prevent="exportMD">Exporter en Markdown</button>

                <hr>

                <span v-if="loading">Chargement...</span>

                <transition name="fade">
                    <div class="alert alert-danger" v-if="error">
                        <button class="close" @click="error = null">&times;</button>
                        {{ error }}
                    </div>
                </transition>

                <transition name="fade">
                    <pre v-if="exportJson && commits.length > 0">{{ exportJson }}</pre>
                </transition>

                <transition name="fade">
                    <div v-if="exportMark" class="card">
                        <div class="card-header">
                            <h4 class="float-left">Export en Markdown</h4>
                            <button class="btn btn-default float-right" v-if="previewMode === 'raw'" @click="previewMode = 'mark'">
                                Aperçu
                            </button>
                            <button class="btn btn-default float-right" v-if="previewMode === 'mark'" @click="previewMode = 'raw'">
                                Voir le code markdown
                            </button>
                            <button class="btn btn-default float-right" @click="(mdFormating) ? mdFormating = false : mdFormating = true" style="margin-right: 15px;">
                                Changer le format
                            </button>
                            <button class="btn btn-default float-right" @click="showCommits = true, exportMark = null" style="margin-right: 15px;">
                                Retour à l'édition
                            </button>
                        </div>
                        <div class="card-body">
                            <div v-if="mdFormating">
                                <label for="mdFormat">Format : (Option disponibles: %auteur%, %message%, %date%, %br%)</label>
                                <div class="input-group">
                                    <input v-model="mdFormat" id="mdFormat" type="text" class="form-control">
                                    <button @click.prevent="exportMD(false)" class="input-group-addon">Valider</button>
                                </div>
                                <br>
                            </div>
                            <div v-if="exportMark.length != 0" v-for="mark in exportMark">
                                <div v-html="previewMode === 'mark' ? mark.mark : mark.raw"></div>
                            </div>
                            <div v-if="exportMark.length === 0">
                                Rien à exporter...
                            </div>
                        </div>
                    </div>
                </transition>

                <transition-group name="fade">
                    <div class="card" v-if="showCommits" v-for="(commit, index) in commits" :key="index" style="margin-top: 15px;">
                        <div :style="commit.visible ? null : 'opacity: 0.5;'">
                            <div class="card-header">
                                <img :src="commit.avatar" height="30" width="30" style="border-radius: 3px;">
                                Par {{ commit.author }} | Le {{ commit.date }}
                            </div>
                            <div class="card-body">
                                <div class="card-text">
                                    <p v-if="editing === null">{{ commit.message }}</p>
                                    <div class="input-group" v-if="editing === commits.indexOf(commit)">
                                        <input type="text" class="form-control" v-model="edition" @keyup.enter="updateMessage(commit)" :placeholder="commit.message">
                                        <span class="input-group-addon" @click="updateMessage(commit)">Editer</span>
                                    </div>
                                </div>
                                <hr>
                                <a href="#" v-if="commit.visible" @click.prevent="editMessage(commit)" class="card-link">Editer</a>
                                <a href="#" v-if="commit.visible" class="card-link" @click.prevent="toggleVisibility(commit)">Cacher</a>
                                <a href="#" v-if="!commit.visible" class="card-link" @click.prevent="toggleVisibility(commit)">Afficher</a>
                            </div>
                        </div>
                    </div>
                </transition-group>
            </div>
        </div>
    </div>
</template>

<script>
    import Vue from 'vue'
    import marked from 'marked'

    export default {
        name: 'Changelog',
        http: {
            headers: {
                Authorization: 'Basic 0c95343864262b10c327a156301dda7d75c9b8d7'
            }
        },
        data () {
            return {
                commits: [],
                owner: null,
                repository: null,
                error: null,
                loading: false,
                showCommits: true,
                editing: null,
                edition: '',
                availabesBranches: [],
                branchSelected: 'master',
                brancheState: 'Chargement...',
                exportJson: null,
                previewMode: 'mark',
                exportMark: null,
                mdFormating: false,
                mdFormat: '### %message% %br% Par %auteur% | Le %date% %br%%br%',
                thTimer: null,
                thLast: null
            }
        },
        computed: {
            url () {
                return 'https://github.com/' + this.owner + '/' + this.repository
            }
        },
        watch: {
            owner () {
                this.throttle(this.searchBranches(), 1000)
            },
            repository () {
                this.throttle(this.searchBranches(), 1000)
            }
        },
        methods: {
            searchCommits () {
                this.error = null
                this.commits = []
                this.exportJson = null
                this.exportMark = null
                if (this.owner === null || this.repository === null) {
                    this.error = 'Les champs "Utilisateur" et "Dépôt" doivent êtres renseignés...'
                } else {
                    this.loading = true
                    this.showCommits = true
                    let self = this
                    this.$http.get('https://api.github.com/repos/' + this.owner + '/' + this.repository + '/commits?sha=' + this.branchSelected).then(function (response) {
                        self.loading = false
                        response.body.forEach(function (resp) {
                            self.commits.push({
                                author: resp.commit.author.name,
                                date: new Date(resp.commit.author.date).toLocaleDateString('fr-FR'),
                                message: resp.commit.message,
                                avatar: resp.author.avatar_url,
                                visible: true
                            })
                        })
                    }, function () {
                        self.loading = false
                        self.error = 'Dépôt non trouvé ou impossible à joindre...'
                    })
                }
            },
            searchBranches () {
                return function () {
                    if (this.repository != null) {
                        this.brancheState = 'Chargement...'
                        this.$http.get('https://api.github.com/repos/' + this.owner + '/' + this.repository + '/branches').then(function (response) {
                            this.brancheState = null
                            this.branchSelected = response.body[0].name
                            this.availabesBranches = response.body
                        }, function () {
                            this.availabesBranches = []
                            this.brancheState = 'Impossible de récupérer le nom des branches disponibles...'
                        })
                    }
                }
            },
            editMessage (commit) {
                this.editing == null ? this.editing = this.commits.indexOf(commit) : this.editing = null
            },
            updateMessage (commit) {
                if (this.edition !== '' && this.edition !== null) {
                    let index = this.commits.indexOf(commit)
                    this.commits[index].message = this.edition
                }
                this.editing = null
                this.edition = null
            },
            toggleVisibility (commit) {
                this.editing = null
                let index = this.commits.indexOf(commit)
                Vue.set(this.commits[index], 'visible', !this.commits[index].visible)
            },
            reset () {
                this.commits = []
                this.owner = null
                this.repository = null
                this.error = null
                this.editing = null
            },
            exportJSON () {
                if (this.commits.length > 0) {
                    if (this.exportJson == null) {
                        this.showCommits = false
                        let exportJson = []
                        this.commits.forEach(function (commit) {
                            if (commit.visible) {
                                commit = Object.assign({}, commit)
                                delete commit.visible
                                exportJson.push(JSON.stringify(commit))
                            }
                        })
                        this.exportJson = exportJson
                    } else {
                        this.exportJson = null
                    }
                } else {
                    this.error = 'Impossible d\'exporter les données sans avoir préalablement affiché les commits...'
                }
            },
            exportMD (toggleHide) {
                if (this.commits.length > 0) {
                    this.showCommits = false
                    if (this.exportMark === null || toggleHide === false) {
                        let exportMark = []
                        let self = this
                        this.commits.forEach(function (commit) {
                            if (commit.visible) {
                                let raw = self.mdFormat.replace('%message%', commit.message).replace('%auteur%', commit.author).replace('%date%', commit.date).replace(/%br%/g, '<br />')
                                let mark = marked(raw.replace(/<br \/>/g, '\n'))
                                exportMark.push({
                                    raw: raw,
                                    mark: mark
                                })
                            }
                        })
                        this.exportMark = exportMark
                    } else {
                        this.exportMark = null
                    }
                } else {
                    this.error = 'Impossible d\'exporter les données sans avoir préalablement affiché les commits...'
                }
            },
            throttle (callback, delay) {
                var context = this
                var now = +new Date()
                var args = arguments
                if (this.thLast && now < this.thLast + delay) {
                    clearTimeout(this.thTimer)
                    this.thTimer = setTimeout(function () {
                        this.thLast = now
                        callback.apply(context, args)
                    }, delay)
                } else {
                    this.thLast = now
                    callback.apply(context, args)
                }
            }
        }
    }
</script>

<style>
    .fade-enter-active, .fade-leave-active {
        transition: opacity .2s
    }

    .fade-enter, .fade-leave-to {
        opacity: 0
    }
</style>