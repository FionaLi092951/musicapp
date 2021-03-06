<template>
    <div class="player" v-show="playList.length > 0">
        <transition name="normal"
        @enter="enter"
        @after-enter="afterEnter"
        @leave="leave"
        @after-leave="afterLeave">
            <div class="normal-player" v-show="fullScreen">
                <div class="background">
                    <img width="100%" height="100%" :src="currentSong.image">
                </div>
                <div class="top">
                    <div class="back" @click="back">
                        <i class="icon-back"></i>
                    </div>
                    <h1 class="title" v-html="currentSong.name"></h1>
                    <h2 class="subtitle" v-html="currentSong.singer"></h2>
                </div>
                <div class="middle">
                    <div class="middle-l" ref="middleL">
                        <div class="cd-wrapper" ref="cdWrapper">
                            <div  class="cd" ref="imageWrapper">
                                <img ref="image" class="image" :class="cdCls" :src="currentSong.image">
                            </div>
                        </div>
                        <div class="playing-lyric-wrapper">
                            <div class="playing-lyric">{{playingLyric}}</div>
                        </div>
                    </div>
                    <scroll class="middle-r" ref="lyricList" :data="currentLyric && currentLyric.lines">歌词显示</scroll>
                </div>
                <div class="bottom">
                    <div class="dot-wrapper">
                        <span class="dot" :class="{'active': currentShow === 'cd'}"></span>
                        <span class="dot" :class="{'active': currentShow === 'lyric'}"></span>
                    </div>
                    <div class="progress-wrapper">
                        <span class="time time-l">{{formatTime(currentTime)}}</span>
                        <div class="progress-bar-wrapper">
                            <progress-bar :percent="percent" @percentChange="onProgressBarChange"></progress-bar>
                        </div>
                        <span class="time time-r">{{formatTime(currentSong.duration)}}</span>
                    </div>
                    <div class="operators">
                        <div class="icon i-left">
                            <i :class="iconMode"></i>
                        </div>
                        <div class="icon i-left" :class="disableCls" @click="prev">
                            <i class="icon-prev"></i>
                        </div>
                        <div class="icon i-center" :class="disableCls">
                            <i @click="togglePlaying" :class="playIcon"></i>
                        </div>
                        <div class="icon i-right" :class="disableCls" @click="next">
                            <i class="icon-next"></i>
                        </div>
                        <div class="icon i-right">
                            <i class="icon" :class="favoriteIcon"></i>
                        </div>
                    </div>
                </div>
            </div>
        </transition>
        <transition name="mini">
            <div class="mini-player" v-show="!fullScreen" @click="open">
                <div class="icon">
                    <div class="imgWrapper" ref="miniWrapper">
                        <img :src="currentSong.image" width="40" height="40" :class="cdCls" class="image">
                    </div>
                </div>
                <div class="text">
                    <h2 class="name" v-html="currentSong.name"></h2>
                    <p class="desc" v-html="currentSong.singer"></p>
                </div>
                <div class="control">
                    <progress-circle radius="30" :percent="percent">
                        <i @click.stop="togglePlaying" :class="miniIcon"></i>
                    </progress-circle>
                </div>
                <div class="control" @click.stop="showPlaylist">
                    <i class="icon-playlist"></i>
                </div>
            </div>
        </transition>
        <audio ref="audio" :src="currentSong.url" @playing="ready" @error="error" @timeupdate="updateTime"></audio>
    </div>
</template>

<script>
import {mapGetters, mapMutations} from 'vuex'
import animations from 'create-keyframe-animation'
import {prefixStyle} from 'common/js/dom'
import ProgressBar from 'base/progress-bar/progress-bar'
import ProgressCircle from 'base/progress-circle/progress-circle'
const transform = prefixStyle('transform')
// const transitionDuration = prefixStyle('transitionDuration')

export default {
    data() {
        return {
            playingLyric: '正在播放的歌词',
            currentShow: 'cd',
            songReady: false, // audio是否已经加载准备播放
            currentTime: 0
        }
    },
    computed: {
        // 播放或暂停
        cdCls() {
            return this.playing ? 'play' : 'icon-pause'
        },
        disableCls() {
            return this.songReady ? '' : 'disable'
        },
        playIcon() {
            return this.playing ? 'icon-pause' : 'icon-play'
        },
        miniIcon() {
            return this.playing ? 'icon-pause-mini' : 'icon-play-mini'
        },
        percent() {
            return this.currentTime / this.currentSong.duration
        },
        ...mapGetters([
            'fullScreen',
            'playList',
            'currentSong',
            'playing',
            'currentIndex'
        ])
    },
    watch: {
        currentSong() {
            this.$nextTick(() => {
                this.$refs.audio.play()
            })
        },
        playing(newPlaying) {
            const audio = this.$refs.audio
            this.$nextTick(() => {
                newPlaying ? audio.play() : audio.pause()
            })
        }
    },
    methods: {
        // 歌曲加载ready时触发
        ready() {
            // 监听 playing 这个事件可以确保慢网速或者快速切换歌曲导致的 DOM Exception
            this.songReady = true
        },
        // 歌曲加载失败
        error() {
            // 保证某一首歌有问题无法加载时不影响之后的播放
            this.songReady = true
        },
        // 歌曲播放进度更新
        updateTime(e) {
            // 获取audio当前播放的时间
            this.currentTime = e.target.currentTime
        },
        // 下一曲
        next() {
            if (!this.songReady) {
                return
            }
            let index = this.currentIndex + 1
            if (index === this.playList.length) {
                index = 0
            }
            this.setCurrentIndex(index)
            if (!this.playing) {
                this.togglePlaying()
            }
            // songReady会在这一首歌曲被加载成功时改成true
            this.songReady = false
        },
        // 上一曲
        prev() {
            if (!this.songReady) {
                return
            }
            let index = this.currentIndex - 1
            if (index === -1) {
                index = this.playList.length - 1
            }
            this.setCurrentIndex(index)
            if (!this.playing) {
                this.togglePlaying()
            }
            this.songReady = false
        },
        // 暂停 & 播放
        togglePlaying() {
            if (!this.songReady) {
                return
            }
            this.setPlayingState(!this.playing)
        },
        onProgressBarChange(percent) {
            this.$refs.audio.currentTime = this.currentSong.duration * percent
            if (!this.playing) {
                this.togglePlaying()
            }
        },
        // 缩小播放器
        back() {
            this.setFullScreen(false)
        },
        // 全屏显示播放器
        open() {
            this.setFullScreen(true)
        },
        // 全屏播放器打开时的动画
        enter(el, done) {
            // 定义动画
            const {x, y, scale} = this._getPosAndScale()
            let animation = {
                0: {
                    transform: `translate3d(${x}px,${y}px,0) scale(${scale})`
                },
                60: {
                    transform: `translate3d(0,0,0) scale(1.1)`
                },
                100: {
                    transform: `translate3d(0,0,0) scale(1)`
                }
            }
            // 注册动画
            animations.registerAnimation({
                name: 'move',
                animation,
                presets: {
                    duration: 400,
                    easing: 'linear'
                }
            })
            // 在Element上运行动画，运行结束后调用done回调
            animations.runAnimation(this.$refs.cdWrapper, 'move', done)
        },
        afterEnter() {
            // 清空animation
            animations.unregisterAnimation('move')
            this.$refs.cdWrapper.style.animation = ''
        },
        leave(el, done) {
            this.$refs.cdWrapper.style.transition = 'all 0.4s'
            const {x, y, scale} = this._getPosAndScale()
            this.$refs.cdWrapper.style[transform] = `translate3d(${x}px,${y}px,0) scale(${scale})`
            const timer = setTimeout(done, 400)
            this.$refs.cdWrapper.addEventListener('transitionend', () => {
                clearTimeout(timer)
                done()
            })
        },
        afterLeave() {
            this.$refs.cdWrapper.style.transition = ''
            this.$refs.cdWrapper.style[transform] = ''
        },
        formatTime(interval) {
            interval = Math.floor(interval)
            const minute = Math.floor(interval / 60)
            const second = interval % 60
            return `${minute}:${this._pad(second)}`
        },
        // 给num部位，默认补成2位，比如：6 -> 06
        _pad(num, n = 2) {
            let len = num.toString().length
            while (len < n) {
                num = '0' + num
                len++
            }
            return num
        },
        _getPosAndScale() {
            // 左下角的位置
            const targetWidth = 40
            const paddingLeft = 40
            const paddingBottom = 30
            // 大图的位置
            const paddingTop = 80
            const width = window.innerWidth * 0.8
            // 缩放比例
            const scale = targetWidth / width
            const x = -(window.innerWidth / 2 - paddingLeft)
            const y = window.innerHeight - paddingTop - width / 2 - paddingBottom
            return {
                x,
                y,
                scale
            }
        },
        ...mapMutations({
            setFullScreen: 'SET_FULL_SCREEN',
            setPlayingState: 'SET_PLAYING_STATE',
            setCurrentIndex: 'SET_CURRENT_INDEX'
        })
    },
    components: {
        ProgressBar,
        ProgressCircle
    }
}
</script>

<style scoped lang="stylus" rel="stylesheet/stylus">
@import "~common/stylus/variable"
@import "~common/stylus/mixin"

.player
    .normal-player
        position: fixed;
        top: 0;
        bottom: 0;
        left: 0;
        right: 0;
        z-index: 150;
        background: $color-background
        .background
            position: absolute;
            left: 0;
            top: 0;
            width: 100%;
            height: 100%;
            z-index: -1;
            opacity: 0.6;
            filter: blur(20px)
        .top
            position: relative;
            margin-bottom: 25px;
            .back
                position: absolute;
                top: 0;
                left: 6px;
                z-index: 50;
                .icon-back
                    display: block;
                    padding: 9px;
                    font-size: $font-size-large-x
                    color: $color-theme
                    transform: rotate(-90deg)
            .title
                width: 70%
                margin: 0 auto
                line-height: 40px
                text-align: center
                no-wrap()
                font-size: $font-size-large
                color: $color-text
            .subtitle
                line-height: 20px
                text-align: center
                font-size: $font-size-medium
                color: $color-text
        .middle
            position: fixed
            width: 100%
            top: 80px
            bottom: 170px
            white-space: nowrap
            font-size: 0
            .middle-l
                display: inline-block
                vertical-align: top
                position: relative
                width: 100%
                height: 0
                padding-top: 80%
                position: absolute
                .cd-wrapper
                    left: 10%
                    top: 0
                    width: 80%
                    box-sizing: border-box
                    height: 100%
                    position: absolute
                    .cd
                        width: 100%
                        height: 100%
                        border-radius: 50%
                        .image
                            position: absolute
                            left: 0
                            top: 0
                            width: 100%
                            height: 100%
                            box-sizing: border-box
                            border-radius: 50%
                            border: 10px solid rgba(255, 255, 255, 0.1)
                        .play
                            animation: rotate 10s linear infinite
                        .icon-pause
                            animation: rotate 10s linear infinite
                            animation-play-state: paused
                .playing-lyric-wrapper
                    width: 80%
                    margin: 30px auto 0 auto
                    overflow: hidden
                    text-align: center
                    .playing-lyric
                        height: 20px
                        line-height: 20px
                        font-size: $font-size-medium
                        color: $color-text-l
        .bottom
            position: absolute
            bottom: 50px
            width: 100%
            .dot-wrapper
                text-align: center
                font-size: 0
                .dot
                    display: inline-block
                    vertical-align: middle
                    margin: 0 4px
                    width: 8px
                    height: 8px
                    border-radius: 50%
                    background: $color-text-l
                    &.active
                        width: 20px
                        border-radius: 5px
                        background: $color-text-ll
            .progress-wrapper
                display: flex
                align-items: center
                width: 80%
                margin: 0px auto
                padding: 10px 0
                .time
                    color: $color-text
                    font-size: $font-size-small
                    flex: 0 0 30px
                    line-height: 30px
                    width: 30px
                    &.time-l
                      text-align: left
                    &.time-r
                      text-align: right
                .progress-bar-wrapper
                    flex: 1
            .operators
                display: flex
                align-items: center
                .icon
                    flex: 1
                    color: $color-theme
                    &.disable
                        color: $color-theme-d
                    i
                        font-size: 30px
                .i-left
                    text-align: right
                .i-center
                    padding: 0 20px
                    text-align: center
                    i
                        font-size: 40px
                .i-right
                    text-align: left
                .icon-favorite
                    color: $color-sub-theme
        &.normal-enter-active, &.normal-leave-active
            transition: all 0.4s
            .top, .bottom
                transition all 0.4s cubic-bezier(0.86, 0.18, 0.82, 1.32)
        &.normal-enter, &.normal-leave-to
            opacity: 0
            .top
                transform: translate3d(0, -100px, 0)
            .bottom
                transform: translate3d(0, 100px, 0)
    .mini-player
        position: fixed;
        left: 0;
        bottom: 0;
        width: 100%;
        height: 60px;
        background: $color-highlight-background;
        z-index: 180;
        display: flex;
        align-items: center;
        .icon
            width: 40px;
            height: 40px;
            padding: 0 10px 0 20px;
            .imgWrapper
                height: 100%;
                width: 100%;
                .image
                    border-radius: 50%;
                .play
                    animation: rotate 10s linear infinite
                .icon-pause
                    animation: rotate 10s linear infinite
                    animation-play-state: paused
        .text
            flex: 1;
            display: flex;
            flex-direction: column;
            justify-content: center;
            line-height: 20px;
            overflow: hidden;
            .name
                margin-bottom: 2px;
                no-wrap()
                font-size: $font-size-medium
                color: $color-text
            .desc
                no-wrap()
                font-size: $font-size-small
                color: $color-text-d
        .control
            width: 30px;
            padding: 0 10px;
            .icon-playlist, .icon-play-mini, .icon-pause-mini
                font-size: 30px;
                color: $color-theme-d
            .icon-play-mini, .icon-pause-mini
                position: absolute
                left: 0
                top: 0
                z-index: -1
        &.mini-enter-active, &.mini-leave-active
            transition: all 0.4s
        &.mini-enter, &.mini-leave-active
            opacity: 0
    @keyframes rotate
        0%
            transform: rotate(0)
        100%
            transform: rotate(360deg)
</style>
