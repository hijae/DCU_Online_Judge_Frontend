<template>
  <div class="setting-main">
    <div class="section-title">{{$t('m.Avatar_Setting')}}</div>
    <template v-if="!avatarOption.imgSrc">
      <Upload type="drag"
              class="mini-container"
              accept=".jpg,.jpeg,.png,.bmp,.gif"
              action=""
              :before-upload="handleSelectFile">
        <div style="padding: 30px 0">
          <Icon type="ios-cloud-upload" size="52" style="color: #3399ff"></Icon>
          <p>이미지 파일을 여기로 드롭하거나, 이곳을 클릭하세요</p>
        </div>
      </Upload>
    </template>

    <template v-else>
      <div class="flex-container">
        <div class="cropper-main inline">
          <vueCropper
            ref="cropper"
            autoCrop
            fixed
            :autoCropWidth="200"
            :autoCropHeight="200"
            :img="avatarOption.imgSrc"
            :outputSize="avatarOption.size"
            :outputType="avatarOption.outputType"
            :info="true"
            @realTime="realTime">
          </vueCropper>
        </div>
        <ButtonGroup vertical class="cropper-btn">
          <Button @click="rotate('left')">
            <Icon type="arrow-return-left" size="20"></Icon>
          </Button>
          <Button @click="rotate('right')">
            <Icon type="arrow-return-right" size="20"></Icon>
          </Button>
          <Button @click="reselect">
            <Icon type="refresh" size="20"></Icon>
          </Button>
          <Button @click="finishCrop">
            <Icon type="checkmark-round" size="20"></Icon>
          </Button>
        </ButtonGroup>
        <div class="cropper-preview" :style="previewStyle">
          <div :style=" preview.div">
            <img :src="avatarOption.imgSrc" :style="preview.img">
          </div>
        </div>
      </div>
    </template>
    <Modal v-model="uploadModalVisible"
           title="아바타 업로드하기">
      <div class="upload-modal">
        <img :src="uploadImgSrc"/>
      </div>
      <div slot="footer">
        <Button @click="uploadAvatar" :loading="loadingUploadBtn">업로드</Button>
      </div>
    </Modal>

    <div class="section-title">{{$t('m.Profile_Setting')}}</div>
    <Form ref="formProfile" :model="formProfile">
      <Row type="flex" :gutter="30" justify="space-around">
        <Col :span="11">
          <FormItem label="실명">
            <Input v-model="formProfile.realname"/>
          </FormItem>
          <FormItem label="언어/Language/語言/语言">
            <Select v-model="formProfile.language">
              <Option v-for="lang in languages" :key="lang.value" :value="lang.value">{{lang.label}}</Option>
            </Select>
          </FormItem>
          <Form-item>
            <Button type="primary" @click="updateProfile" :loading="loadingSaveBtn">저장하기</Button>
          </Form-item>
        </Col>

        <Col :span="11">
          <Form-item label="학번">
            <Input v-model="formProfile.schoolssn"/>
          </Form-item>
        </Col>
      </Row>
    </Form>
  </div>
</template>

<script>
  import api from '@oj/api'
  import utils from '@/utils/utils'
  import {VueCropper} from 'vue-cropper'
  import {types} from '@/store'
  import {languages} from '@/i18n'

  export default {
    components: {
      VueCropper
    },
    data () {
      return {
        loadingSaveBtn: false,
        loadingUploadBtn: false,
        uploadModalVisible: false,
        preview: {},
        uploadImgSrc: '',
        avatarOption: {
          imgSrc: '',
          size: 0.8,
          outputType: 'png'
        },
        languages: languages,
        formProfile: {
          realname: '',
          schoolssn: ''
        }
      }
    },
    mounted () {
      let profile = this.$store.state.user.profile.user
      console.log(profile)
      Object.keys(this.formProfile).forEach(element => {
        if (profile[element] !== undefined) {
          this.formProfile[element] = profile[element]
        }
      })
    },
    methods: {
      checkFileType (file) {
        if (!/\.(gif|jpg|jpeg|png|bmp|GIF|JPG|PNG)$/.test(file.name)) {
          this.$Notice.warning({
            title: '지원되지 않는 형식의 파일입니다.',
            desc: '파일 ' + file.name + ' 의 형식이 올바르지 않습니다. 이미지파일만 업로드 해주세요.'
          })
          return false
        }
        return true
      },
      checkFileSize (file) {
        // max size is 2MB
        if (file.size > 2 * 1024 * 1024) {
          this.$Notice.warning({
            title: '파일의 최대 사이즈를 초과하였습니다.',
            desc: '파일 ' + file.name + ' 이(가) 너무 큽니다, 2MB이내의 크기의 이미지파일을 업로드 해주세요.'
          })
          return false
        }
        return true
      },
      handleSelectFile (file) {
        let isOk = this.checkFileType(file) && this.checkFileSize(file)
        if (!isOk) {
          return false
        }
        let reader = new window.FileReader()
        reader.onload = (e) => {
          this.avatarOption.imgSrc = e.target.result
        }
        reader.readAsDataURL(file)
        return false
      },
      realTime (data) {
        this.preview = data
      },
      rotate (direction) {
        if (direction === 'left') {
          this.$refs.cropper.rotateLeft()
        } else {
          this.$refs.cropper.rotateRight()
        }
      },
      reselect () {
        this.$Modal.confirm({
          content: '변경사항을 취소하시겠습니까?',
          onOk: () => {
            this.avatarOption.imgSrc = ''
          }
        })
      },
      finishCrop () {
        this.$refs.cropper.getCropData(data => {
          this.uploadImgSrc = data
          this.uploadModalVisible = true
        })
      },
      uploadAvatar () {
        this.$refs.cropper.getCropBlob(blob => {
          let form = new window.FormData()
          let file = new window.File([blob], 'avatar.' + this.avatarOption.outputType)
          form.append('image', file)
          this.loadingUploadBtn = true
          this.$http({
            method: 'post',
            url: 'upload_avatar',
            data: form,
            headers: {'content-type': 'multipart/form-data'}
          }).then(res => {
            this.loadingUploadBtn = false
            this.$success('성공적으로 새 아바타를 설정하였습니다.')
            this.uploadModalVisible = false
            this.avatarOption.imgSrc = ''
            this.$store.dispatch('getProfile')
          }, () => {
            this.loadingUploadBtn = false
          })
        })
      },
      updateProfile () {
        this.loadingSaveBtn = true
        let updateData = utils.filterEmptyValue(Object.assign({}, this.formProfile))
        console.log(updateData)
        api.updateProfile(updateData).then(res => {
          this.$success('Success')
          this.$store.commit(types.CHANGE_PROFILE, {profile: res.data.data})
          this.loadingSaveBtn = false
        }, _ => {
          this.loadingSaveBtn = false
        })
      }
    },
    computed: {
      previewStyle () {
        return {
          'width': this.preview.w + 'px',
          'height': this.preview.h + 'px',
          'overflow': 'hidden'
        }
      }
    }
  }
</script>

<style lang="less" scoped>
  .inline {
    display: inline-block;
  }

  .copper-img {
    width: 400px;
    height: 300px;
  }

  .flex-container {
    flex-wrap: wrap;
    justify-content: flex-start;
    margin-bottom: 10px;
    .cropper-main {
      flex: none;
      .copper-img;
    }
    .cropper-btn {
      flex: none;
      vertical-align: top;
    }
    .cropper-preview {
      flex: none;
      /*margin: 10px;*/
      margin-left: 20px;
      box-shadow: 0 0 1px 0;
      .copper-img;
    }
  }

  .upload-modal {
    .notice {
      font-size: 16px;
      display: inline-block;
      vertical-align: top;
      padding: 10px;
      padding-right: 15px;
    }
    img {
      box-shadow: 0 0 1px 0;
      border-radius: 50%;
    }
  }
</style>
