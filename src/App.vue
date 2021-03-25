<template>
  <div class="box grid column justify-center items-center">
    <div class="title">CDA Downloader</div>
    <div v-if="!displayVideoBox && !loading" class="form">
      <input type="text" placeholder="Adres URL" class="url" v-model="url" />
      <button v-on:click="download()">
        <i class="fas fa-cloud-download-alt download"></i>
      </button>
    </div>
    <div v-show="displayVideoBox && !loading" class="video-box grid column">
      <div
        v-on:click="close()"
        class="close grid row items-center justify-center"
      >
        <i class="fas fa-times"></i>
      </div>
      <div class="grid row items-center">
        <div class="video">
          <img v-if="!playVideo" class="thumb" :src="image" />
          <span v-on:click="play()" v-if="!playVideo" class="play-button">
            <i class="fas fa-play"></i>
          </span>
          <video id="video" :src="urlVideo" v-if="playVideo" controls autoplay />
        </div>
        <div class="grid column info">
          <h3>{{ title }}</h3>
          <span class="grid row items-center">
            <h4>Pobierz:</h4>
            <button
              v-for="q in qualities"
              v-bind:key="q"
              class="download"
              :class="{ active: include('quality-btn-active', q.class) }"
            >
              {{ q.title }}
            </button>
          </span>
          <a :href="urlVideo">{{ urlVideo }}</a>
        </div>
      </div>
    </div>

    <div v-if="loading" class="loading grid column justify-center">
      <div class="grid row items-center justify-center">
        <h3>Loading</h3>
      </div>
    </div>
    <div class="copyright">
      <span></span><br />
      Wykonał: Mateusz Henicz
    </div>
  </div>
</template>


<script lang="ts">
import { Options, Vue } from "vue-class-component";
import HelloWorld from "./components/HelloWorld.vue";
// @ts-ignore
import artoo from "artoo-js";
// @ts-ignore
import cheerio from "cheerio";


@Options({
  components: {
    HelloWorld,
  },
  data() {
    return {
      loading: false,
      url: "",
      displayVideoBox: false,
      playVideo: false,
      title: "",
      image: "",
      qualities: [],
      urlVideo: "",
    };
  },
  mounted() {
    artoo.bootstrap(cheerio);
  },
  methods: {
    findBetween(strToParse: string, strStart: string, strFinish: string) {
      var str = strToParse.match(strStart + "(.*?)" + strFinish);
      if (str === null) return "";
      if (str.length < 2) return "";
      return str[1];
    },

    rot13(str: string) {
      var input = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz";
      var output = "NOPQRSTUVWXYZABCDEFGHIJKLMnopqrstuvwxyzabcdefghijklm";
      // @ts-ignore
      var index = (x) => input.indexOf(x);
      // @ts-ignore
      var translate = (x) => (index(x) > -1 ? output[index(x)] : x);
      return str.split("").map(translate).join("");
    },

    ma(a: string) {
      a = a.replace(".cda.mp4", "");
      a = a.replace(".2cda.pl", ".cda.pl");
      a = a.replace(".3cda.pl", ".cda.pl");
      return "https://" + a + ".mp4";
    },

    ia(a: string) {
      for (var b = [], e = 0; e < a.length; e++) {
        var f = a.charCodeAt(e);
        b[e] =
          33 <= f && 126 >= f
            ? String.fromCharCode(33 + ((f + 14) % 94))
            : String.fromCharCode(f);
      }
      return this.ma(b.join(""));
    },

    ja(a: string) {
      let c;
      String.fromCharCode(
        ("Z" >= a ? 82 : 132) >= (c = a.charCodeAt(0) + 11) ? c : c - 55
      );
      return this.ia(a);
    },

    la(a: string) {
      return decodeURIComponent(a);
    },

    L(a: string) {
      return a.replace(/[a-zA-Z]/g, function (a: any) {
        return String.fromCharCode(
          ("Z" >= a ? 90 : 122) >= (a = a.charCodeAt(0) + 13) ? a : a - 26
        );
      });
    },

    ka(a: string) {
      return this.ja(this.la(this.L(a)));
    },

    M(a: string) {
      let c;
      String.fromCharCode(
        ("Z" >= a ? 11 : 344) >= (c = a.charCodeAt(0) + 22) ? c : c - 11
      );
      a = a.replace("_XDDD", "");
      a = a.replace("_CDA", "");
      a = a.replace("_ADC", "");
      a = a.replace("_CXD", "");
      a = a.replace("_QWE", "");
      a = a.replace("_Q5", "");
      a = a.replace("_IKSDE", "");
      a = this.ka(this.L(a));
      return a;
    },

    include: function (one: string, two: string) {
      return two.includes(one);
    },
    close: function () {
      this.loading = false;
      this.url = "";
      this.displayVideoBox = false;
      this.title = "";
      this.image = "";
      this.qualities = [];
    },
    regex: function (html: string) {
      const match = /(\+x5\()\'(?<hash>(.)*)(\'\)\+)/.exec(html);
      return match!.groups!.hash;
    },
    play() {
      this.playVideo = true;
    },
    download: function () {
      this.loading = true;
      fetch("https://pobieracz.net/?URL=" + this.url)
        .then((res) => res.text())
        .then((html) => {
          console.log(html);
          let movieURL = this.regex(html);

          // in 2020 urls use M()
          if (this.M(movieURL) == this.M("")) {
            throw "empty movieURL";
          }
          if (this.M(movieURL).includes(".mp4")) {
            console.log("input:\t", movieURL);
            movieURL = this.M(movieURL);
          }

          console.log("v.url:\t", movieURL);

          this.urlVideo = movieURL;

          fetch(this.url)
            .then((res) => res.text())
            .then((htmlTwo) => {
              var $ = cheerio.load(htmlTwo);

              artoo.setContext($);

              this.title = artoo.scrape("title")[0];

              this.image = artoo.scrape("img", {
                src: "src",
                title: "title",
              })[1].src;

              this.qualities = JSON.parse(
                JSON.stringify(
                  artoo.scrape(".wrapqualitybtn a", {
                    url: "href",
                    title: "text",
                    class: "class",
                  })
                )
              );

              console.log("q", this.qualities);

              this.displayVideoBox = true;
              this.loading = false;
            });
        });
    },
  },
})
export default class App extends Vue {}
</script>

<style lang="scss">
body,
html {
  font-family: "Open Sans";
  background: linear-gradient(-45deg, #ee7752, #e73c7e, #23a6d5, #23d5ab);
  background-size: 400% 400%;
  -webkit-animation: Gradient 15s ease infinite;
  -moz-animation: Gradient 15s ease infinite;
  animation: Gradient 15s ease infinite;
  width: 100%;
  height: 100%;
  overflow: hidden;
}

.grid {
  display: -webkit-box;
  display: -ms-flexbox;
  display: flex;
}

.row {
  -webkit-box-orient: horizontal;
  -webkit-box-direction: normal;
  -ms-flex-direction: row;
  flex-direction: row;
}

.column {
  -webkit-box-orient: vertical;
  -webkit-box-direction: normal;
  -ms-flex-direction: column;
  flex-direction: column;
}

.items-center {
  align-items: center;
}

.justify-center {
  justify-content: center;
}

.loading {
  width: 200px;
  height: 200px;
  background: white;
  border-radius: 10px;
  padding: 10px;
  color: black;
  text-align: center;
  h3 {
    margin: 0;
  }
}

.video-box {
  position: relative;
  width: 1000px;
  background: white;
  height: fit-content;
  border-radius: 10px;
  padding: 10px;
  color: black;
  word-break: break-all;
  .close {
    position: absolute;
    height: 30px;
    width: 30px;
    top: 0;
    right: 0;
    cursor: pointer;
  }
  .video {
    width: 100%;
    margin-right: 15px;
    border-radius: 10px;
    position: relative;
    .thumb {
      width: 100%;
      height: 100%;
      border-radius: 10px;
    }
    video {
      width: 100%;
      height: 100%;
      border-radius: 10px;
    }
    .play-button {
      position: absolute;
      top: 50%;
      left: 50%;
      transform: translate(-50%, -50%);
      i {
        font-size: 35px;
        cursor: pointer;
        color: white;
        transition: color 0.3s ease;
      }
      i:hover {
        color: rgb(207, 197, 197);
      }
    }
  }

  .info {
    width: 100%;
    height: 100%;
    padding-left: 20px;
    h3 {
      margin: 0;
    }

    span {
      margin-top: 38px;
      h4 {
        margin: 0;
        margin-right: 10px;
      }
      .active {
        background: chocolate !important;
      }
      .download {
        width: fit-content;
        border: 0;
        background: #149274;
        background-size: 400% 400%;
        -webkit-animation: Gradient 15s ease infinite;
        animation: Gradient 15s ease infinite;
        padding: 10px;
        color: white;
        border-radius: 5px;
        font-family: "Open Sans";
        font-weight: 600;
        margin-right: 5px;
        cursor: pointer;
        transition: background 0.3s ease;
      }
      .download:hover {
        background: #136582;
      }
    }
  }
}

a {
  text-decoration: none;
}

.box {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  color: white;
}

.title {
  color: white;
  font-size: 30px;
  text-align: center;
  margin-bottom: 20px;
}
.copyright {
  color: white;
  font-size: 15px;
  text-align: center;
  margin-top: 20px;
}

.form {
  display: flex;
  align-items: center;
  justify-content: center;
  background: linear-gradient(-90deg, #964932, #821e44, #136582, #149274);
  background-size: 400% 400%;
  -webkit-animation: Gradient 15s ease infinite;
  -moz-animation: Gradient 15s ease infinite;
  animation: Gradient 15s ease infinite;
}

.success {
  display: none;
  background: #27ae60;
  text-align: center;
  padding: 10px;
  width: 351px;
}

.error {
  display: none;
  background: #c0392b;
  text-align: center;
  padding: 10px;
  width: 351px;
}

input[type="text"] {
  color: white;
  width: 300px;
  height: 50px;
  background: transparent;
  border: none;
  font-size: 20px;
  float: left;
  padding-left: 25px;
  -webkit-border-radius: 5px;
  -moz-border-radius: 5px;
  border-radius: 5px;
  outline: inherit;
}

::placeholder {
  color: white;
  opacity: 1;
}

.form button {
  -webkit-border-top-right-radius: 5px;
  -webkit-border-bottom-right-radius: 5px;
  -moz-border-radius-topright: 5px;
  -moz-border-radius-bottomright: 5px;
  border-top-right-radius: 5px;
  border-bottom-right-radius: 5px;
  border: none;
  background: transparent;
  height: 52px;
  width: 50px;
  font-size: 10pt;
  -webkit-transition: all 0.55s ease;
  -moz-transition: all 0.55s ease;
  -ms-transition: all 0.55s ease;
  -o-transition: all 0.55s ease;
  transition: all 0.55s ease;
  transition: all 0.55s ease;
  margin-left: -4px;
  outline: inherit;
  color: white;
  cursor: pointer;
}

@-webkit-keyframes Gradient {
  0% {
    background-position: 0% 50%;
  }
  50% {
    background-position: 100% 50%;
  }
  100% {
    background-position: 0% 50%;
  }
}

@-moz-keyframes Gradient {
  0% {
    background-position: 0% 50%;
  }
  50% {
    background-position: 100% 50%;
  }
  100% {
    background-position: 0% 50%;
  }
}

@keyframes Gradient {
  0% {
    background-position: 0% 50%;
  }
  50% {
    background-position: 100% 50%;
  }
  100% {
    background-position: 0% 50%;
  }
}
</style>
