<template>
  <div class="container">
    <h1>{{ msg }}</h1>
    <button v-on:click="fetchAuth" type="button" name="button">Sign in</button>

    <div class="row">
      <div class="col-md-6" v-for='item in tracksLists.items' track-by="$index">
       <article>
         <h2>{{ item.year }}</h2>
           <div class="row">
         <div class="col-md-3" v-for='track in item.tracks' track-by="$index">
            <h3>{{ $index }}</h3>
            <p>{{ track.name }}</p>
            <p>{{ track.album.year }}</p>
          </div>
       </article>
     </div>
    </div>
  </div>
</template>

<script>
import Q from 'q'
import Spotify from 'spotify-web-api-js'

let config = {
  stateKey: 'spotify_auth_state',
  queryParams: {
    client_id: '831eb2150ccc4193b32ea7a2a82dd9b0',
    client_secret: 'd3b19ac433c84174aae210703cb5edc6',
    response_type: 'token',
    redirect_uri: 'http://localhost:8080/',
    scope: 'playlist-read-private playlist-modify-public user-follow-read playlist-read-collaborative',
    state: ''
  },
  redirectUrl: 'https://accounts.spotify.com/authorize?',
  limitAlbumsRequest: 20,
  limitPlaylistTracksRequest: 100
}

export default {
  data () {
    return {
      msg: 'Hello World!',
      artistAlbums: {
        albums: [],
        albumsIds: []
      },
      tracksLists: {},
      spotifyApi: new Spotify(),
      config: config,
      userAuthenticated: false
    }
  },
  methods: {
    fetchAuth: function () {
      window.location.href = config.redirectUrl +
      'client_id=' + config.queryParams.client_id +
      '&response_type=' + config.queryParams.response_type +
      '&redirect_uri=' + config.queryParams.redirect_uri +
      '&scope=' + config.queryParams.scope +
      '&state=' + config.queryParams.state
    },

    verifyAuth: function () {
      let params = this.getHashParams()
      let accessToken = params.access_token
      let state = params.state

      if (accessToken && (state == null)) {
        console.log('auth failed')
        return false
      } else {
        if (accessToken) {
          this.userAuthenticated = true
          this.spotifyApi.setAccessToken(accessToken)
          return true
        } else {
          console.log('auth failed 2')
          return false
        }
      }
    },

    getHashParams: function () {
      let hashParams = {}
      let e
      let i = 0
      const r = /([^&;=]+)=?([^&;]*)/g
      const q = window.location.hash.substring(1)
      while (i < 4) {
        e = r.exec(q)
        hashParams[e[1]] = decodeURIComponent(e[2])
        i++
      }
      return hashParams
    },
    getPlaylistTracks: function () {
      let self = this
      let tracksAlbumids = []
      let tracks = []
      let total = 0
      this.msg = 'Loading artist albums'
      this.spotifyApi.setPromiseImplementation(Q)
      this.spotifyApi.getPlaylistTracks('117725775', '048T2fuVyNIJJXrflJsEk0',
        {
          market: 'BE',
          limit: config.limitPlaylistTracksRequest,
          fields: 'total,items(track(name,id,album(id)))'
        })
        .then(function (data) {
          total = data.total
          console.log(total)

          var cptTotal = 0
          var fetchLimit = Math.floor(total / config.limitPlaylistTracksRequest)
          var remainder = 0
          if (total % config.limitPlaylistTracksRequest !== 0) {
            fetchLimit++
            remainder = total % config.limitPlaylistTracksRequest
          }
          while (cptTotal < fetchLimit) {
            let offset = cptTotal + config.limitPlaylistTracksRequest
            self.spotifyApi.getPlaylistTracks('117725775', '048T2fuVyNIJJXrflJsEk0',
              {
                market: 'BE',
                limit: cptTotal === fetchLimit - 1 ? config.limitPlaylistTracksRequest : remainder,
                fields: 'total,items(track(name,id,album(id)))',
                offset: offset
              })
            .then(function (datab) {
              datab.items.forEach(function (track) {
                var myTrack = {
                  name: track.track.name,
                  id: track.track.id,
                  album: {
                    id: track.track.album.id,
                    year: ''
                  }
                }
                tracks.push(myTrack)
                tracksAlbumids.push(track.track.album.id)
              })
              let tracksYears = []
              self.msg = 'Loading years'
              var cpt = 0
              // const bite = tracksAlbumids
              console.log(tracksAlbumids.length)
              while (cpt < data.total) {
                var currentLimit = cpt + config.limitAlbumsRequest
                // var dafuq = bite.slice(cpt, currentLimit)
                // console.log('cpt : ' + cpt)
                // console.log(dafuq)
                // console.log('limit: ' + currentLimit)
                // console.log('??? : ' + bite.length)
                // get albums by ids (limit 20)
                self.spotifyApi.getAlbums(tracksAlbumids.slice(cpt, currentLimit), {market: 'BE'})
                .then(function (data) {
                  let responseAlbums = data.albums
                  for (let i = 0; i < config.limitAlbumsRequest; i++) {
                    tracksYears.push(responseAlbums[i].release_date.substring(0, 4))
                    tracks[i].album.year = (responseAlbums[i].release_date.substring(0, 4))
                  }
                  // return fetched albums
                  var returnArray = {
                    items: [],
                    yearlist: []
                  }

                  for (var i = 0; i < tracksYears.length; i++) {
                    var currentYear = tracksYears[i]
                    var yearPlaylist = []

                    if (returnArray.items.length === 0 || returnArray.items.length !== 0 && returnArray.yearlist.indexOf(currentYear) === -1) {
                      yearPlaylist = {
                        year: currentYear,
                        tracks: []
                      }
                      returnArray.items.push(yearPlaylist)
                      returnArray.yearlist.push(currentYear)
                    }
                  }
                  for (var j = 0; j < returnArray.items.length; j++) {
                    for (var k = 0; k < tracks.length; k++) {
                      if (returnArray.items[j].year === tracks[k].album.year) {
                        returnArray.items[j].tracks.push(tracks[k])
                      }
                    }
                  }
                  console.log(returnArray)
                  self.tracksLists = returnArray
                })
                .catch(function (error) {
                  console.error(error)
                })
                cpt = cpt + config.limitAlbumsRequest
              } // end while retrieving albums
            }).catch(function (error) {
              console.error(error)
            })
            cptTotal++
          } // end while get tracks
        }).catch(function (error) {
          console.error(error)
        }) // end get total tracks
    }
  },
  ready: function () {
    if (this.verifyAuth()) {
      this.getPlaylistTracks()
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
h1 {
  color: #42b983;
}


</style>
