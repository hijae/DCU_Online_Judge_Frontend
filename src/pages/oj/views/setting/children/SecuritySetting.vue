<template>
  <div class="setting-main">
    <p class="section-title">{{$t('m.Sessions')}}</p>
    <div class="flex-container setting-content">
      <template v-for="session in sessions">
        <Card :padding="20" class="flex-child">
          <span slot="title" style="line-height: 20px">{{session.ip}}</span>
          <div slot="extra">
            <Tag v-if="session.current_session" color="green">현재</Tag>
            <Button v-else
                    type="warning"
                    size="small"
                    @click="deleteSession(session.session_key)">삭제
            </Button>
          </div>
          <Form :label-width="100">
            <FormItem label="운영체제 :" class="item">
              {{session.user_agent | platform}}
            </FormItem>
            <FormItem label="브라우저 :" class="item">
              {{session.user_agent | browser}}
            </FormItem>
            <FormItem label="마지막 활동 :" class="item">
              {{session.last_activity | localtime }}
            </FormItem>
          </Form>
        </Card>

      </template>
    </div>

    <p class="section-title">{{$t('m.Two_Factor_Authentication')}}</p>
    <div class="mini-container setting-content">
      <Form>
        <Alert v-if="TFAOpened"
               type="success"
               class="notice"
               showIcon>You have enabled two-factor authentication.
        </Alert>
        <FormItem v-if="!TFAOpened">
          <div class="oj-relative">
            <img :src="qrcodeSrc" id="qr-img">
            <Spin size="large" fix v-if="loadingQRcode"></Spin>
          </div>
        </FormItem>
        <template v-if="!loadingQRcode">
          <FormItem style="width: 250px">
            <Input v-model="formTwoFactor.code" placeholder="이중 인증을 위해 QR코드를 인식하세요."/>
          </FormItem>
          <Button type="primary"
                  :loading="loadingBtn"
                  @click="updateTFA(false)"
                  v-if="!TFAOpened">Open TFA
          </Button>
          <Button type="error"
                  :loading="loadingBtn"
                  @click="closeTFA"
                  v-else>Close TFA
          </Button>
        </template>
      </Form>
    </div>
  </div>
</template>

<script>
  import api from '@oj/api'
  import {mapGetters, mapActions} from 'vuex'
  import browserDetector from 'browser-detect'

  const browsers = {}
  const loadBrowser = (userAgent) => {
    let browser = {}
    if (userAgent in Object.keys(browsers)) {
      browser = browsers[userAgent]
    } else {
      browser = browserDetector(userAgent)
      browsers[userAgent] = browser
    }
    return browser
  }

  export default {
    data () {
      return {
        qrcodeSrc: '',
        loadingQRcode: false,
        loadingBtn: false,
        formTwoFactor: {
          code: ''
        },
        sessions: []
      }
    },
    mounted () {
      this.getSessions()
      if (!this.TFAOpened) {
        this.getAuthImg()
      }
    },
    methods: {
      ...mapActions(['getProfile']),
      getAuthImg () {
        this.loadingQRcode = true
        api.twoFactorAuth('get').then(res => {
          this.loadingQRcode = false
          this.qrcodeSrc = res.data.data
        })
      },
      getSessions () {
        api.getSessions().then(res => {
          let data = res.data.data
          // 将当前session放到第一个
          let sessions = data.filter(session => {
            return session.current_session
          })
          data.forEach(session => {
            if (!session.current_session) {
              sessions.push(session)
            }
          })
          this.sessions = sessions
        })
      },
      deleteSession (sessionKey) {
        this.$Modal.confirm({
          title: '확인',
          content: '세션을 삭제하시겠습니까?',
          onOk: () => {
            api.deleteSession(sessionKey).then(res => {
              this.getSessions()
            }, _ => {
            })
          }
        })
      },
      closeTFA () {
        this.$Modal.confirm({
          title: '확인',
          content: '이중 인증은 계정을 보호하는 강력한 도구입니다. 닫으시겠습니까?',
          onOk: () => {
            this.updateTFA(true)
          }
        })
      },
      updateTFA (close) {
        let method = close === false ? 'post' : 'put'
        this.loadingBtn = true
        api.twoFactorAuth(method, this.formTwoFactor).then(res => {
          this.loadingBtn = false
          this.getProfile()
          if (close === true) {
            this.getAuthImg()
            this.formTwoFactor.code = ''
          }
          this.formTwoFactor.code = ''
        }, err => {
          this.formTwoFactor.code = ''
          this.loadingBtn = false
          if (err.data.data.indexOf('session') > -1) {
            this.getProfile()
            this.getAuthImg()
          }
        })
      }
    },
    computed: {
      ...mapGetters(['user']),
      TFAOpened () {
        return this.user && this.user.two_factor_auth
      }
    },
    filters: {
      browser (value) {
        let b = loadBrowser(value)
        if (b.name && b.version) {
          return b.name + ' ' + b.version
        } else {
          return 'Unknown'
        }
      },
      platform (value) {
        let b = loadBrowser(value)
        return b.os ? b.os : 'Unknown'
      }
    }
  }
</script>

<style lang="less" scoped>
  .notice {
    font-size: 16px;
    margin-bottom: 20px;
    display: inline-block;
  }

  .oj-relative {
    width: 150px;
    #qr-img {
      width: 300px;
      margin: -10px 0 -30px -20px;
    }
  }

  .flex-container {
    flex-flow: row wrap;
    justify-content: flex-start;
    .flex-child {
      flex: 1 0;
      max-width: 350px;
      margin-right: 30px;
      margin-bottom: 30px;
      .item {
        margin-bottom: 0;
      }
    }
  }
</style>
