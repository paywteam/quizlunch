<template>
  <div class="container">
    <div class="title">
      <div class="logo" />uizLunch
    </div>
    <div class="quiz-area">
      <div class="quiz-wrapper">
        <div class="top">
          <div class="qt-title"> <!--{{ quiz.title }}--> </div>
          <transition name="fade" mode="out-in">
            <div class="qt-money" v-if="!isChange">
              <div class="qt-money-int">{{ quizMoneyInt }}.</div>
              <div class="qt-money-float">{{ quizMoneyFloat }}</div>
              <div class="qt-money-unit">&#8361;</div>
            </div>
          </transition>
        </div>
        <div class="middle">
          <div class="quiz-move">
            <div class="quiz-left" v-if="!isFirst" v-on:click="previousQuiz();" />
          </div>
          <div class="qm-content-wrapper" >
            <transition name="fade" mode="out-in">
              <div class="qm-content" v-if="!isChange">{{ quiz.information }}</div>
            </transition>
          </div>
          <div class="quiz-move">
            <div class="quiz-right" v-if="!isLast" v-on:click="nextQuiz();"/>
          </div>
        </div>
        <div class="bottom">
          - made by {{ quiz.author }}
        </div>
      </div>
    </div>
    <div class="answer-area">
      <form class="a-input" v-if="quiz.answer === ''" v-on:submit.prevent="postAnswer();">
        <input class="ai-textarea" type="textarea" v-model="answerTextarea" required maxlength="80">
        <div class="input-cushion" />
        <input class="ai-submit" type="submit" value="Enter">
      </form>
      <div class="a-solved" v-else v-on:click="answerCovered=!answerCovered">
        <transition name="fade" mode="out-in">
          <div class="as-cover" v-if="answerCovered === true" key="cover">
            show
          </div>
          <div class="as-answer" v-else key="notCover">
            {{ quiz.answer }}
          </div>
        </transition>
      </div>
    </div>
    <div class="comment-area">
      <form class="c-input" v-if="isLast" v-on:submit.prevent="postComment();">
        <input class="ci-textarea" type="textarea" v-model="commentTextarea" required minlength="2" maxlength="80">
        <div class="input-cushion" />
        <input class="ci-password" type="password" v-model="commentPassword" required minlength="4">
        <div class="input-cushion" />
        <input class="ci-submit" type="submit" value="Enter">
      </form> 
      <div class="c-container" v-infinite-scroll="moreComments" infinite-scroll-disabled="busy" infinite-scroll-distance="1">
        <transition-group name="list-fade" tag="p">
          <div class="cc-comment" v-for="comment in comments" v-bind:key="comment.commentID">
            <div class="comment-info">
              <div class="comment-nickname">{{ comment.nickname }}</div>
              <div class="comment-time">{{ comment.time }}</div>
              <div class="comment-ip">{{ comment.ip }}</div>
              <div class="comment-delete">
                <input class="cd-button" type="submit" value="" @click="checkPassword(comment.commentID)" />
              </div>
            </div>
            <div class="comment-context">{{ comment.text }}</div>
          </div>
        </transition-group>
        <div class='loader' v-if="loading"/> 
      </div>
    </div>
  </div>
</template>

<script>
import axios from 'axios'

export default {
  name: 'quizlunch_main',
  data() {
    return {
      ws: '',
      baseURL: {
        db : 'https://db.api.quizlunch.com',
        rng : 'https://rng.api.quizlunch.com',
        db_ws: 'wss://db2.api.quizlunch.com'
      },
      quiz: {},
      answer: '',
      answerCovered: true,
      comments: [],
      numOfComments: 20,
      commentTextarea: '',
      commentPassword: '',
      answerTextarea: '',
      deletePassword: '',
      busy: true,
      loading: false,
      isFirst: false,
      isLast: true,
      isChange: false
    }
  },
  computed: {
    quizMoneyInt(){
      if(this.quiz.money)
        return parseInt(this.quiz.money)
    },
    quizMoneyFloat(){
      if(this.quiz.money){
        var intLen = parseInt(this.quiz.money).toString().length
        return ((this.quiz.money).toFixed(3)).toString().slice(intLen+1)
      }
    }
  },
  mounted(){
    this.initWS()
    this.calcMoney()
    setTimeout(()=>{
      this.busy = false
    },1000)
  },
  methods: {
    checkPassword(commentID) {
      const password = window.prompt('password를 입력해주세요')
      this.deleteComment(commentID,password)
    },
    calcMoney(){
      var tick = 33
      var totalSecond = 24*60*60
      var totalPrize = totalSecond/20
      var moneyPerSecond = totalPrize / totalSecond
      var moneyPerTick = parseFloat(moneyPerSecond/(1000/tick))
      setInterval(()=>{
        if(this.isLast === true && this.quiz.gotAnswer === 0)
          this.quiz.money += moneyPerTick
      }, tick)
    },
    //
    // WebSocket
    //
    async initWS(){
      this.ws = new WebSocket(this.baseURL['db_ws'])
      this.onOpenWS()
      this.onMessageWS()
      this.onErrorWS()
      this.onCloseWS()
    },
    async onOpenWS(){
      this.ws.onopen = (event)=>{
        console.log('connected')
      }
    },
    //
    // Websocket
    //
    async onMessageWS(){
      this.ws.onmessage = (event)=>{
        var result = JSON.parse(event.data)
        if(result['renew comments']){
          this.comments = result['renew comments']
        }
        if(result['insert comment']){
          if(this.quiz.quizID === result['insert comment'].quizID){
            this.comments = [result['insert comment']].concat(this.comments)
            this.comments = this.comments.slice(0,this.numOfComments)
          }
        }
        if(result['delete comment']){
          for(var i=0; i<this.comments.length; i++){
            if(this.comments[i]['commentID'] == result['delete comment'])
            {
              this.comments.splice(i, 1)
              break;
            }
          }
        }
        if(result['renew quiz']){
          this.isChange = true;
          this.quiz = result['renew quiz'];
          setTimeout(()=>{
            this.isChange = false
          },300)
        }
        if(result['renew money']){
          if(this.quiz['quizID'] === result['renew money']['quizID'] && this.quiz['gotAnswer'] === 0)
            this.quiz.money = result['renew money']['value']
        }
      }
    },
    async onErrorWS(){
      this.ws.onerror = (event)=>{
        console.log("Sever error message :" + event.data )
      }
    },
    async onCloseWS(){
      this.ws.onclose = (event)=>{
        console.log("Sever closed")
        setTimeout(()=>{
          this.initWS()
        },2000)
      }
    },
    //
    // REST API
    //
    async postComment(){
      if(this.quiz.quizID === undefined) // quiz is not exist
        return

      if(this.commentTextarea.trim().length < 2) // comment is too shorts
        return
    
      const url = `${this.baseURL['db']}/comment`
      var body = {}
      body['quizID'] = this.quiz.quizID 
      body['text'] = this.commentTextarea
      body['password'] = this.commentPassword
      this.commentTextarea = ''

      await axios.post(url, body)
    },
    async deleteComment(commentID,password){
      if(password === null)
        return

      const url = `${this.baseURL['db']}/comment`
      var data = {}
      data['commentID'] = commentID
      data['password'] = password

      var result = await axios.delete(url,{data})
      if(result.data !== 200){
        alert("비밀번호가 틀립니다.");
      }
      
    },
    async moreComments(){
      if(this.quiz.quizID === undefined)
        return
      if(this.busy === true)
        return
      
      this.busy = true

      const url = `${this.baseURL['db']}/comment/more`
      var body = {params:{}}
      body['params']['quizID'] = this.quiz.quizID
      body['params']['numOfComments'] = this.numOfComments

      this.loading = true
      var result = await axios.get(url, body)
      this.loading = false

      if(result.data.length !== 0){
        this.comments = this.comments.concat(result.data)
        this.numOfComments += 20
        this.busy = false
      }
      else{
        setTimeout(()=>{
          this.busy = false
        },10000)
      }
    },
    async postAnswer(){
      if(this.quiz.quizID === undefined)
        return
      
      const url = `${this.baseURL['db']}/quiz/${this.quiz.quizID}/${this.answerTextarea}`
      var result = await axios.get(url)
      if(result.data === 200){
        location.href = 'https://quizlunch.com/awards'
      }
      else{
        alert(`${result.data.count}명의 사용자가 오답 '${this.answerTextarea}'을 시도했습니다.`)
      }
      this.answerTextarea = ''
    },
    async previousQuiz(){
      if(this.quiz.quizID === undefined)
        return
      
      const url = `${this.baseURL['db']}/quiz/${this.quiz.quizID}/previous`
      
      var result = await axios.get(url)
      if(result.data){
        this.isChange = true;
        this.quiz = result.data.quiz
        this.comments = result.data.comments
        this.isFirst = result.data.isFirst
        this.isLast = result.data.isLast
        setTimeout(()=>{
          this.isChange = false
        },300)
      }
    },
    async nextQuiz(){
      if(this.quiz.quizID === undefined)
        return

      const url = `${this.baseURL['db']}/quiz/${this.quiz.quizID}/next`
      
      var result = await axios.get(url)
      if(result.data){
        this.isChange = true;
        this.quiz = result.data.quiz
        this.comments = result.data.comments
        this.isFirst = result.data.isFirst
        this.isLast = result.data.isLast
        setTimeout(()=>{
          this.isChange = false
        },300)
      }

    }
  }
}
</script>

<style lang="scss">

.container {
  // padding: 0 3rem;
  .title {
    padding: 1.5rem 0;
    text-align: center; //dev
    font-size: 2.5rem;
    font-weight: 600;
    .logo {
      display: inline-block;
      width: 2.5rem;
      height: 2.5rem;
      margin-right: 0.1rem;
      background: url('~assets/img/logo.svg') no-repeat;
    }
  }

  .quiz-area {
    
    display: flex;
    align-content: center;
    padding-bottom: 0.5rem;

    .quiz-wrapper {
      
      flex: auto;
      display: flex;
      flex-direction: column;

      text-align: center;

      margin: 0 0.5rem;
      border-radius: 1rem;
      background: #EEF1F6;
      background-size: cover;
      box-shadow: 1px 1px 2px rgba(0, 0, 0, 0.15);
      .top {
        flex-basis: 2rem;
        display: flex;

        padding: 0.5rem 0;
        margin: 0 1rem;
        
        .qt-title {
          flex: auto;
          text-align: center;
        }
        .qt-money {
          flex-basis: 6rem;

          font-size: 0;
          text-align: right;
          & * {
            font-size: initial;
          }
          
          .qt-money-int{
            display: inline-block;
          }
          
          .qt-money-float{
            display: inline-block;
            width: 1.5rem;
            text-align: left;
            font-size: 0.85rem;
          }

          .qt-money-unit{
            display: inline-block;
            padding-left: 0.3rem;
          }
        }
      }
      
      .middle {
        flex: auto;
        display: flex;
        
        align-items: center;
        .quiz-move {
          flex-basis: 2.5rem;
          height:2.5rem;
          width:2.5rem;
          
          .quiz-left {
            width: inherit;
            height: inherit;
            background: url('~assets/img/left.png') no-repeat;
            background-size: cover;
          }
          .quiz-right {
            width: inherit;
            height: inherit;
            background: url('~assets/img/right.png') no-repeat;
            background-size: cover;
          }
        }
        .qm-content-wrapper {
          flex: auto;
          display: flex;
          min-height: 10rem;
          align-items: center;
          justify-content: center;

          .qm-content {
            white-space:pre-wrap;
          }

        }
      }

      .bottom {
        flex-basis: 1rem;

        padding: 0.5rem 0;
        margin: 0 1rem;

        color: #999;
        font-size: 0.5rem;
        text-align: right;
      } // bottom
    } // quiz-wrapper
  } // quiz-area

  .answer-area{
    display: flex;
    justify-content: center;

    padding-bottom: 2rem;
    .a-input {
      display: flex;
      width: 9rem;
      .ai-textarea {
        flex: auto;
        min-width: 0; // override min-width: auto
        
        padding-left: 0.5rem;
        border: none;
        border-radius: 1rem 0 0 1rem;

        font-size: 1rem;
        font-family: inherit;
        background: #EEF1F6;
        outline: none;
        box-shadow: 1px 1px 2px rgba(0, 0, 0, 0.15);
        -webkit-appearance: none; // safari default input style
      }

      .ai-submit {
        flex-basis: 3rem;
        
        padding-left:0.3rem;
        padding-right:0.5rem;
        border: none;
        border-radius: 0 1rem 1rem 0;

        text-align: center;
        font-size: 0.8rem;
        background:none;
        background: #EEF1F6;
        outline: none;
        box-shadow: 1px 1px 2px rgba(0, 0, 0, 0.15);
        -webkit-appearance: none; // safari default input style
      }

    } //a-input
    .a-solved{
      min-width: 9rem;
      padding: 0.1rem 0.5rem;
      border: none;
      border-radius: 1rem;

      text-align: center;
      font-size: 1rem;
      background:none;
      background-color: #EEF1F6;
      outline: none;
      box-shadow: 1px 1px 2px rgba(0, 0, 0, 0.15);
      .as-cover{
        //
      }
      .as-answer{
        text-align: center;
      }
    }
  } // answer-area

  .comment-area {

    font-size: 0.75rem;
    .c-input {
      display: flex;
      
      margin: 0 0.5rem;
      .ci-textarea {
        flex: auto;
        min-width: 0; // override min-width: auto
        
        padding-left: 0.5rem;
        border: none;
        border-radius: 1rem 0 0 1rem;

        font-size: 1rem;
        font-family: inherit;
        background: #EEF1F6;
        outline: none;
        box-shadow: 1px 1px 2px rgba(0, 0, 0, 0.15);
        -webkit-appearance: none; // safari default input style
      }

      .ci-password {
        flex-basis: 3rem;
        min-width: 0; // override min-width: auto
        
        padding: 0 0.25rem;
        border: none;
        border-radius: 0;

        background: #EEF1F6;
        font-size: 1rem;
        outline: none;
        box-shadow: 1px 1px 2px rgba(0, 0, 0, 0.15);
        -webkit-appearance: none; // safari default input style
      }

      .ci-submit {
        flex-basis: 3rem;
        
        padding-left:0.3rem;
        padding-right:0.5rem;
        border: none;
        border-radius: 0 1rem 1rem 0;

        text-align: center;
        font-size: 0.8rem;
        background:none;
        background-color: #EEF1F6;
        outline: none;
        box-shadow: 1px 1px 2px rgba(0, 0, 0, 0.15);
        -webkit-appearance: none; // safari default input style
      }

    }

    .c-container {
      margin: 0 0.5rem;
      .cc-comment {
        padding: 0.3rem 0;
        margin-top: 0.5rem;
        border-radius: 10px;

        align-items: center;
        background: #d3dae6;
        box-shadow: 1px 1px 2px rgba(0, 0, 0, 0.15);
        transition: all 0.3s;
        &.list-fade-enter-active {
          transition-delay: 0.3s;
        }
        &.list-fade-enter,
        &.list-fade-leave-to {
          opacity: 0;
        }
        .comment-info {
          display: flex;
          .comment-nickname {
            flex-basis: auto;
            padding: 0 0.5rem;
            font-weight: 400;
          }
          .comment-time {
            flex: auto;

            font-size: 0.7rem;
            color: #888;
            text-align: left;
          }
          .comment-ip {
            flex-basis: 6rem;
            color: #D3DAE6;
            font-size: 0.7rem;
            padding: 0 0.5rem;
            text-align: right;
          }
          .comment-delete {
            flex-basis: 1.6rem;
            
            .cd-button {
              width: 1rem;
              height: 1rem;
              border: none;
              padding: 0;
              background: url('~assets/img/cancel.png') no-repeat;
              background-size: cover;
              border-radius: 0;
            }

          }
        }
        .comment-context {
          flex: 5;
          padding: 0 0.5rem;
          text-align: left;
        } // comment-context
      } // cc-comment
    } // c-container
  } //comment-area

  .input-cushion {
    border-left: 0.5px solid #BFC7D5;
    border-right: 0.5px solid #BFC7D5;
  }
  .loader {
    margin: 1rem auto;
    border: 0.2rem solid #f3f3f3; /* Light grey */
    border-top: 0.2rem solid #000000; /* black */
    border-radius: 50%;
    width: 1.5rem;
    height: 1.5rem;
    animation: spin 1s linear infinite;
  }

  @keyframes spin {
    0% { transform: rotate(0deg); }
    100% { transform: rotate(360deg); }
  }

  .fade-enter-active, .fade-leave-active {
    transition: all 0.3s;
  }
  .fade-enter, .fade-leave-to {
    opacity: 0;
  }

}
</style>
  
