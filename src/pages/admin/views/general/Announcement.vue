<template>
  <div class="announcement view">
    <Panel :title="$t('m.General_Announcement')">
      <div class="list">
        <el-table
          v-loading="loading"
          element-loading-text="loading"
          ref="table"
          :data="announcementList"
          style="width: 100%">
          <el-table-column
            width="100"
            prop="id"
            label="ID">
          </el-table-column>
          <el-table-column
            prop="title"
            label="제목">
          </el-table-column>
          <el-table-column
            prop="create_time"
            label="작성일">
            <template slot-scope="scope">
              {{ scope.row.create_time | localtime }}
            </template>
          </el-table-column>
          <el-table-column
            prop="last_update_time"
            label="마지막 수정 시간">
            <template slot-scope="scope">
              {{scope.row.last_update_time | localtime }}
            </template>
          </el-table-column>
          <el-table-column
            prop="created_by.username"
            label="작성자">
          </el-table-column>
          <el-table-column
            width="100"
            prop="visible"
            label="공개여부">
            <template slot-scope="scope">
              <el-switch v-model="scope.row.visible"
                         active-text=""
                         inactive-text=""
                         @change="handleVisibleSwitch(scope.row)">
              </el-switch>
            </template>
          </el-table-column>
          <el-table-column
            fixed="right"
            label="선택사항"
            width="200">
            <div slot-scope="scope">
              <icon-btn name="편집" icon="edit" @click.native="openAnnouncementDialog(scope.row.id)"></icon-btn>
              <icon-btn name="삭제" icon="trash" @click.native="deleteAnnouncement(scope.row.id)"></icon-btn>
            </div>
          </el-table-column>
        </el-table>
        <div class="panel-options">
          <el-button type="primary" size="small" @click="openAnnouncementDialog(null)" icon="el-icon-plus">새 글 작성 </el-button>
          <el-pagination
            v-if="!contestID"
            class="page"
            layout="prev, pager, next"
            @current-change="currentChange"
            :page-size="pageSize"
            :total="total">
          </el-pagination>
        </div>
      </div>
    </Panel>
    <!--대화상자-->
    <el-dialog :title="announcementDialogTitle" :visible.sync="showEditAnnouncementDialog"
               @open="onOpenEditDialog" :close-on-click-modal="false">
      <el-form label-position="top">
        <el-form-item :label="$t('m.Announcement_Title')" required>
          <el-input
            v-model="announcement.title"
            :placeholder="$t('m.Announcement_Title')" class="title-input">
          </el-input>
        </el-form-item>
        <el-form-item :label="$t('m.Announcement_Content')" required>
          <Simditor v-model="announcement.content"></Simditor>
        </el-form-item>
        <div class="visible-box">
          <span>{{$t('m.Announcement_visible')}}</span>
          <el-switch
            v-model="announcement.visible"
            active-text=""
            inactive-text="">
          </el-switch>
        </div>
      </el-form>
      <span slot="footer" class="dialog-footer">
          <cancel @click.native="showEditAnnouncementDialog = false"></cancel>
          <save type="primary" @click.native="submitAnnouncement"></save>
        </span>
    </el-dialog>
  </div>
</template>

<script>
  import Simditor from '../../components/Simditor.vue'
  import api from '../../api.js'

  export default {
    name: 'Announcement',
    components: {
      Simditor
    },
    data () {
      return {
        contestID: '',
        // 공지 사항 편집 대화 상자 표시
        showEditAnnouncementDialog: false,
        // 公告列表
        announcementList: [],
        // 一页显示的公告数
        pageSize: 15,
        // 总公告数
        total: 0,
        // 当前公告id
        currentAnnouncementId: null,
        mode: 'create',
        // 公告 (new | edit) model
        announcement: {
          title: '',
          visible: true,
          content: ''
        },
        // 对话框标题
        announcementDialogTitle: 'Edit Announcement',
        // 是否显示loading
        loading: true,
        // 当前页码
        currentPage: 0
      }
    },
    mounted () {
      this.init()
    },
    methods: {
      init () {
        this.contestID = this.$route.params.contestId
        if (this.contestID) {
          this.getContestAnnouncementList()
        } else {
          this.getAnnouncementList(1)
        }
      },
      // test
      // 페이지 번호 콜백 전환
      currentChange (page) {
        this.currentPage = page
        this.getAnnouncementList(page)
      },
      getAnnouncementList (page) {
        this.loading = true
        api.getAnnouncementList((page - 1) * this.pageSize, this.pageSize).then(res => {
          this.loading = false
          this.total = res.data.data.total
          this.announcementList = res.data.data.results
        }, res => {
          this.loading = false
        })
      },
      getContestAnnouncementList () {
        this.loading = true
        api.getContestAnnouncementList(this.contestID).then(res => {
          this.loading = false
          this.announcementList = res.data.data
        }).catch(() => {
          this.loading = false
        })
      },
      // 콜백하여 편집 대화 상자 열기
      onOpenEditDialog () {
        // todo 최적화
        // 일시적으로 해결 文本编辑器显示异常bug
        setTimeout(() => {
          if (document.createEvent) {
            let event = document.createEvent('HTMLEvents')
            event.initEvent('resize', true, true)
            window.dispatchEvent(event)
          } else if (document.createEventObject) {
            window.fireEvent('onresize')
          }
        }, 0)
      },
      // 편집 제출
      // 默认传入MouseEvent
      submitAnnouncement (data = undefined) {
        let funcName = ''
        if (!data.title) {
          data = {
            id: this.currentAnnouncementId,
            title: this.announcement.title,
            content: this.announcement.content,
            visible: this.announcement.visible
          }
        }
        if (this.contestID) {
          data.contest_id = this.contestID
          funcName = this.mode === 'edit' ? 'updateContestAnnouncement' : 'createContestAnnouncement'
        } else {
          funcName = this.mode === 'edit' ? 'updateAnnouncement' : 'createAnnouncement'
        }
        api[funcName](data).then(res => {
          this.showEditAnnouncementDialog = false
          this.init()
        }).catch()
      },
      // 공지 사항 삭제
      deleteAnnouncement (announcementId) {
        this.$confirm('이 공지 사항을 삭제 하시겠습니까?', '경고', {
          confirmButtonText: 'Delete',
          cancelButtonText: 'Cancel',
          type: 'warning'
        }).then(() => {
          // then 为确定
          this.loading = true
          let funcName = this.contestID ? 'deleteContestAnnouncement' : 'deleteAnnouncement'
          api[funcName](announcementId).then(res => {
            this.loading = true
            this.init()
          })
        }).catch(() => {
          // catch 为取消
          this.loading = false
        })
      },
      openAnnouncementDialog (id) {
        this.showEditAnnouncementDialog = true
        if (id !== null) {
          this.currentAnnouncementId = id
          this.announcementDialogTitle = '공지 사항 편집'
          this.announcementList.find(item => {
            if (item.id === this.currentAnnouncementId) {
              this.announcement.title = item.title
              this.announcement.visible = item.visible
              this.announcement.content = item.content
              this.mode = 'edit'
            }
          })
        } else {
          this.announcementDialogTitle = '새 공지 사항 작성'
          this.announcement.title = ''
          this.announcement.visible = true
          this.announcement.content = ''
          this.mode = 'create'
        }
      },
      handleVisibleSwitch (row) {
        this.mode = 'edit'
        this.submitAnnouncement({
          id: row.id,
          title: row.title,
          content: row.content,
          visible: row.visible
        })
      }
    },
    watch: {
      $route () {
        this.init()
      }
    }
  }
</script>

<style lang="less" scoped>
  .title-input {
    margin-bottom: 20px;
  }

  .visible-box {
    margin-top: 10px;
    width: 205px;
    float: left;
  }
</style>
