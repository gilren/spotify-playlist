<template>
  <div class="container">
    <h1>{{ msg }}</h1>
    <button v-on:click="fetchAuth" type="button" name="button">Sign in</button>
    <p>Total Removed tracks : {{ removedTracks.total }}</p>
    <p v-for='item in removedTracks.items' track-by="$index">{{ item }}</p>
    <div class="row">
      <div class="col-md-6" v-for='item in tracksLists.items' track-by="$index">
       <article>
         <h2>{{ item.year }}</h2>
           <div class="row">
         <textarea>
           {{ item.tracks }}
         </textarea>
       </article>
          </div>
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
    response_type: 'token',
    redirect_uri: 'http://localhost:8080/',
    scope: 'playlist-read-private playlist-modify-public user-follow-read playlist-read-collaborative',
    state: ''
  },
  delayBetweenAjaxCalls: 600,
  bigDelayBetweenAjaxCalls: 3600,
  limitAlbumsRequest: 20,
  limitPlaylistTracksRequest: 100,
  redirectUrl: 'https://accounts.spotify.com/authorize?',
  userId: '117725775',
  playlistId: '5XZwxq9GFaNiKwTKsz41Kr'
}
export default {
  data () {
    return {
      msg: 'Hello World!',
      tracksLists: {
        items: [],
        yearList: []
      },
      removedTracks: {
        items: [],
        total: 0
      },
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
      let tracks = []
      let total = 0
      let tracksAlbumids = []
      let offset = 0
      let deleted = 0
      this.msg = 'Loading artist albums'
      this.spotifyApi.setPromiseImplementation(Q)
      this.spotifyApi.getPlaylistTracks(config.userId, config.playlistId,
        {
          market: 'BE',
          limit: config.limitPlaylistTracksRequest,
          fields: 'total,items(track(name,id,album(id)))'
        })
        .then(function (data) {
          total = data.total

          var settingsTracks = self.getIterationandRemainder(total, config.limitPlaylistTracksRequest)

          for (var x = 0, ln = settingsTracks.iterations; x < ln; x++) {
            setTimeout(function (y) {
              console.log(' ')
              console.log('%c ' + y + ' FIRST LOOP : ', 'background: blue; color:#fff')

              var tracksLimit = config.limitPlaylistTracksRequest
              offset = y * config.limitPlaylistTracksRequest

              if (settingsTracks.remainder > 0 && y === ln - 1) {
                tracksLimit = tracksLimit - (config.limitPlaylistTracksRequest - settingsTracks.remainder)
              }
              self.spotifyApi.getPlaylistTracks(config.userId, config.playlistId,
                {
                  market: 'BE',
                  limit: tracksLimit,
                  fields: 'total,items(track(name,id,uri,album(id)))',
                  offset: offset
                }
              ).then(function (data) {
                // retrieve data by 100 or remainder
                data.items.forEach(function (track) {
                  if (track.track.album.id !== null) {
                    var myTrack = {
                      name: track.track.name,
                      id: track.track.id,
                      uri: track.track.uri,
                      album: {
                        id: track.track.album.id,
                        year: ''
                      }
                    }
                    tracks.push(myTrack)
                    tracksAlbumids.push(track.track.album.id)
                  } else {
                    console.log('## track deleted : ' + track.track.name)
                    self.removedTracks.items.push([track.track.name])
                    self.removedTracks.total = deleted++
                    // tracksLimit--
                    // offset--
                  }
                }) // end foreach tracks

                var settingsAlbum = self.getIterationandRemainder(tracksLimit, config.limitAlbumsRequest)
                // start loop album ids
                var currentOffset = 0
                var currentLimit = config.limitAlbumsRequest

                for (var a = 0, lo = settingsAlbum.iterations; a < lo; a++) {
                  setTimeout(function (z) {
                    currentOffset = z * config.limitAlbumsRequest
                    currentLimit = z * config.limitAlbumsRequest + config.limitAlbumsRequest
                    // console.log(z)
                    if (settingsAlbum.remainder > 0 && y === ln - 1 && z === lo - 1) {
                      // currentOffset = currentOffset
                      currentLimit = currentLimit - (config.limitAlbumsRequest - settingsAlbum.remainder)
                    }

                    console.log(' ')
                    console.log('%c - ' + z + ' SECOND LOOP ', 'background: #000; color:#fff')
                    console.log('%c --- ' + currentOffset + ' --- ' + currentLimit, 'background: red; color:#fafafa')

                    self.spotifyApi.getAlbums(tracksAlbumids.slice(offset + currentOffset, offset + currentLimit),
                      {
                        market: 'BE'
                      }
                    ).then(function (data) {
                      let responseAlbums = data.albums
                      for (let i = 0; i < responseAlbums.length; i++) {
                        var currentElement = offset + currentOffset + i
                        var currentYear = responseAlbums[i].release_date.substring(0, 4)
                        var currentTrack = tracks[currentElement]
                        currentTrack.album.year = currentYear

                        if (self.tracksLists.yearList.indexOf(currentYear) > -1) {
                          for (let p = 0; p < self.tracksLists.items.length; p++) {
                            if (self.tracksLists.items[p].year === currentYear) {
                              // console.log('||' + currentElement + '|| pushing --> ' + currentTrack.name)
                              self.tracksLists.items[p].tracks.push(currentTrack)
                            }
                          }
                        } else {
                          // console.log('|| pushing ' + currentYear + ' ||')
                          self.tracksLists.yearList.push(currentYear)
                          var myPlaylist = {
                            year: currentYear,
                            tracks: []
                          }
                          // console.log('||' + currentElement + '|| pushing --> ' + currentTrack.name)
                          myPlaylist.tracks.push(currentTrack)
                          self.tracksLists.items.push(myPlaylist)
                        }

                        // console.log(tracks)
                        // self.tracksLists.items = allPlaylists
                        // self.tracksLists.yearList = self.tracksLists.yearList
                      }
                    }).catch(function (error) {
                      console.error(error)
                    }) // end get album ids
                  }, a * self.config.delayBetweenAjaxCalls, a)
                }
              }).catch(function (error) {
                console.error(error)
              }) // end get total tracks catch
            }, x * self.config.bigDelayBetweenAjaxCalls, x)
          } // end retrieving albums
        // end get total tracks
        }).catch(function (error) {
          console.error(error)
        }) // end get total tracks catch
    },
    getIterationandRemainder: function (total, limit) {
      var out = {
        iterations: Math.floor(total / limit),
        remainder: 0
      }
      if (total % limit !== 0) {
        out.iterations++
        out.remainder = total % limit
      }
      return out
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
