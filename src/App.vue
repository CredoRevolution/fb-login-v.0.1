<template>
  <div id="app" class="app">
    <button v-if="!authorised" class="login-button" @click.prevent="login">
      login
    </button>
    <div v-else class="authorized">
      <button class="get-posts-button" @click.prevent="getPosts">
        getPosts
      </button>
      <div class="bussiness-acc">
        <p>{{ bussinessAccount.name }}</p>
        <img
          :src="bussinessAccount.picture"
          width="50"
          height="50"
          style="border-radius: 50%"
        />
        <p>{{ bussinessAccount.connected_instagram_account }}</p>
        <p
          v-if="this.bussinessAccount.feed && this.bussinessAccount.feed.length"
        >
          {{ bussinessAccount.feed }}
        </p>
        <p v-else="this.bussinessAccount.feed">No posts</p>
      </div>
      <div class="inst-acc">
        <p>{{ instAcc.accUsername }}</p>
        <img
          :src="instAcc.accProfilePic"
          width="200"
          height="200"
          style="border-radius: 50%"
        />
      </div>
      <div v-if="posts.length" class="posts">
        <div v-for="post in posts" :key="post.id" class="post">
          <img :src="post.media_url" width="200" height="200" />
          <p>{{ post.caption }}</p>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
let FB_INITIALIZED = false

const axios = require('axios')

export default {
  name: 'App',
  data() {
    return {
      authorised: false,
      posts: [],
      accessToken: '',
      instAcc: {
        accId: '',
        accUsername: '',
        accProfilePic: '',
      },
      bussinessAccount: {
        accId: '',
        feed: [],
        name: '',
        connected_instagram_account: '',
        picture: '',
      },
      accessTokenBussiness: '',
      bussinessAccountId: '',
    }
  },
  methods: {
    login() {
      if (!FB_INITIALIZED) {
        console.warn('FB.init() must be called before FB.login()')
        return
      }
      FB.login(
        (facebookResponse) => {
          console.log('FB.login', facebookResponse)
          this.statusChangeCallback(facebookResponse)
        },
        {
          scope:
            'public_profile,instagram_basic,pages_show_list,instagram_content_publish',
        }
      )
    },
    statusChangeCallback(facebookResponse) {
      if (facebookResponse.status === 'connected') {
        console.log('connected')
        // Logged into your app and Facebook.
        this.authorised = true
        this.accessToken = facebookResponse.authResponse.accessToken
      } else if (facebookResponse.status === 'not_authorized') {
        console.log('not_authorized')
        // The person is logged into Facebook, but not your app.
        this.authorised = false
      }
    },
    getPosts() {
      if (!this.authorised) {
        console.warn('login() must be called before getPosts()')
        return
      }
      axios
        .get(
          `https://graph.facebook.com/v19.0/me/accounts?access_token=${this.accessToken}`,
          {
            headers: {
              'Content-Type': 'application/json',
            },
          }
        )
        .catch((error) => {
          if (error.response.status === 400) {
            console.error('Bad request: please check the access_token')
          } else {
            console.error(error)
          }
        })

        .then((response) => {
          console.log(response)
          this.bussinessAccountId = response.data.data[0].id
          this.accessTokenBussiness = response.data.data[0].access_token
        })
        .catch((error) => {
          console.error(error)
        })

        .then(() => {
          axios
            .get(
              `https://graph.facebook.com/v19.0/${this.bussinessAccountId}?fields=name,picture,feed,connected_instagram_account&access_token=${this.accessTokenBussiness}`,
              {
                headers: {
                  'Content-Type': 'application/json',
                },
              }
            )
            .then((response) => {
              console.log(response)
              if (response.data.feed) {
                this.bussinessAccount.accId = response.data.id
                this.bussinessAccount.feed = response.data.feed.data
              }
              this.bussinessAccount.name = response.data.name || ''
              this.bussinessAccount.picture =
                (response.data.picture && response.data.picture.data.url) || ''
              this.bussinessAccount.connected_instagram_account =
                (response.data.connected_instagram_account &&
                  response.data.connected_instagram_account.id) ||
                ''
              console.log(this.bussinessAccount)
            })
            .catch((error) => {
              console.error(error)
            })
            .then(() => {
              FB.api(
                `/${this.bussinessAccount.connected_instagram_account}/?fields=username,profile_picture_url,media{media_url}`,
                'GET',
                {},
                (response) => {
                  console.log(response)
                  if (response) {
                    this.instAcc.accId = response.id
                    this.instAcc.accUsername = response.username
                    this.instAcc.accProfilePic = response.profile_picture_url
                    this.posts = response.media.data
                  }
                }
              )
            })
            .then(() => {
              console.log(this.instAcc)
            })
            .catch((error) => {
              console.error(error)
            })
        })
    },
  },
  mounted() {
    window.fbAsyncInit = function () {
      FB.init({
        appId: '1530852871068475',
        cookie: true,
        xfbml: true,
        version: 'v19.0',
      })
      FB_INITIALIZED = true
      FB.AppEvents.logPageView()
    }
    ;(function (d, s, id) {
      var js,
        fjs = d.getElementsByTagName(s)[0]
      if (d.getElementById(id)) {
        return
      }
      js = d.createElement(s)
      js.id = id
      js.src = 'https://connect.facebook.net/en_US/sdk.js'
      fjs.parentNode.insertBefore(js, fjs)
    })(document, 'script', 'facebook-jssdk')
  },
}
</script>

<style lang="scss" scoped>
* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

.app {
  display: flex;
  justify-content: center;
  flex-direction: column;
}

.login-button {
  background: #3b5998;
  border: none;
  border-radius: 3px;
  padding: 10px 20px;
  color: #fff;
  cursor: pointer;
}

.authorized {
  display: flex;
  gap: 10px;
  flex-direction: column;
}

.get-posts-button {
  background: #3b5998;
  border: none;
  border-radius: 3px;
  padding: 10px 20px;
  color: #fff;
  cursor: pointer;
}

.posts {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
  padding: 10px;
  background: #fafafa;
  justify-content: center;
}

.post {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.post img {
  border-radius: 3px;
}

.no-posts {
  text-align: center;
  padding: 10px;
  background: #fafafa;
  border: 1px solid #3b5998;
  border-radius: 3px;
}

.inst-acc {
  display: flex;
  align-items: center;
  gap: 10px;
  flex-direction: column;
  justify-content: center p {
    text-align: center;
    font-size: 30px;
    font-weight: bold;
  }
}
</style>
