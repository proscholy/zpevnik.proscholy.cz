<template>
    <div class="container mt-5">
        <div class="row">
            <div class="col-sm-6 offset-sm-3">
                <div class="card">
                    <div class="card-header">Přihlášení do mého zpěvníku</div>
                    <div class="card-body">
                        <div class="alert alert-primary">
                            K přihlášení lze použít existující Google účet.
                        </div>

                        <div>
                            <a @click="signin">Přihlásit se přes Google</a>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>

<script>
import { GoogleProvider } from '~/plugins/firebase';
import gql from 'graphql-tag';

const fetch_items = gql`
    query($search_params: String, $page: Int, $per_page: Int) {
        song_lyrics_paginated: search_song_lyrics(
            search_params: $search_params
            page: $page
            per_page: $per_page
        ) {
            data {
                id
                name
            }
            paginatorInfo {
                currentPage
                lastPage
                total
                hasMorePages
            }
        }
    }
`;

const fetch_user = gql`
    query {
        user: current_public_user {
            id
            name
            email
            playlists {
                id
                name
                song_lyrics {
                    id
                    name
                }
            }
        }
    }
`;

/**
 * Login component.
 *
 * TODO: frontend
 */
export default {
    data() {
        return {
            user: null,
            token: null,
            // search_string: "",
            new_songbook_name: '',

            isLoggedIn: false
        };
    },

    name: 'login',

    apollo: {
        song_lyrics_paginated: {
            query: fetch_items,
            variables() {
                return {
                    search_params: this.searchParams,
                    page: 1,
                    per_page: 100
                };
            },

            debounce: 200
            // result(result) {
            //     this.$emit("query-loaded", null);
            //     this.enable_more = result.data.song_lyrics_paginated.paginatorInfo.hasMorePages;
            //     this.results_loaded = true;
            // },
        }
    },

    mounted() {
        this.$auth.onAuthStateChanged(this.performUserQuery);
    },

    methods: {
        async performUserQuery(firebaseuser) {
            if (firebaseuser) {
                var token = await firebaseuser.getIdToken();

                this.$apollo
                    .query({
                        query: fetch_user,
                        manual: true,
                        context: {
                            headers: {
                                Authorization: 'Bearer ' + token
                            }
                        }
                    })
                    .then(response => {
                        console.log(response.data.user);
                        this.user = response.data.user;
                        this.isLoggedIn = true;
                    })
                    .catch(exc => {});
            }
        },

        /**
         * Sign into account using Google.
         *
         * @returns {Promise<void>}
         */
        async signin() {
            this.provider = GoogleProvider;

            var user = this.$auth.currentUser;

            let token = '';

            // if (user) {
            //     token = await user.getIdToken();
            //     console.log(token);
            // } else {
            let creds = await this.$auth.signInWithPopup(this.provider);

            console.log({ creds });
            token = await creds.user.getIdToken();
            // }

            console.log({ token });
            // console.log({ me });

            // todo: refresh the page
        }
    },

    computed: {
        searchParams() {
            let query = {
                bool: {
                    must: [],
                    filter: []
                }
            };

            return JSON.stringify({
                sort: ['name_keyword'],
                query: query
            });
        },

        song_lyrics() {
            return this.song_lyrics_paginated
                ? this.song_lyrics_paginated.data
                : [];
        }
    }
};
</script>
