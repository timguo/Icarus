<template>
<setting-base>
    <div v-title>个人信息 - 用户设置 - {{state.config.title}}</div>
    <h3 class="ic-header">个人信息</h3>

    <div class="box">
        <div class="left">
            <div class="lbox">
                <div class="setting-item">
                    <span class="label">Email</span>
                    <div class="line">
                        <span>{{user.email}}</span>
                    </div>
                </div>

                <div class="setting-item">
                    <span class="label">昵称</span>
                    <div class="line">
                        <span>{{user.nickname}}</span>
                    </div>
                </div>

                <div class="setting-item">
                    <span class="label">帐户状态</span>
                    <div class="line">
                        <span>{{state.misc.POST_STATE_TXT[user.state]}}</span>
                    </div>
                </div>

                <div class="setting-item">
                    <span class="label">简介</span>
                    <div class="line">
                        <textarea class="ic-input commentArea" rows="5" placeholder="" v-model="user.biology" maxLength="255"></textarea>
                    </div>
                </div>

                <div class="setting-item">
                    <span class="label">URL</span>
                    <div class="line">
                        <input type="text" class="ic-input commentArea" rows="5" placeholder="" v-model="user.url" />
                    </div>
                </div>

                <div class="setting-item">
                    <span class="label">所在地</span>
                    <div class="line">
                        <input type="text" class="ic-input commentArea" rows="5" placeholder="" v-model="user.location" />
                    </div>
                </div>

                <div class="setting-item" v-if="false">
                    <span class="label">手机</span>
                    <div class="line">
                        <input type="text" class="ic-input commentArea" rows="5" placeholder="" v-model="user.phone" />
                    </div>
                </div>

                <button class="ic-btn primary" :class="[changed && (!updating) ? '' : 'disabled']" @click="updateInfo">{{ updating ? '更新中 ...' : '更新个人信息' }}</button>
            </div>
        </div>
        <div class="right">
            <div class="setting-item">
                <div class="line box-avatar" @click="(atLeastUser) ? state.dialog.userSetAvatar = true : null" >
                    <avatar :is-link="false" :user="state.user" :size="200" class="avatar"></avatar>
                    <button v-if="atLeastUser" class="ic-btn primary btn-upload" style="margin-top: 10px">点此上传新头像</button>
                </div>
            </div>

            <div class="setting-item">
                <span class="label">用户组</span>
                <div class="line">
                    <span>{{state.misc.USER_GROUP_TXT[user.group]}}</span>
                    <template v-if="state.isInactiveUser()">
                        <a class="resend" v-if="sending">正在发送中 ...</a>
                        <a class="resend" v-else href="javascript:void(0)" @click="resendActivationMail">重发激活邮件</a>
                    </template>
                </div>
            </div>

            <div class="setting-item">
                <span class="label">注册时间</span>
                <div class="line">
                    <ic-time :ago="false" :timestamp="user.time" />
                </div>
            </div>

            <div class="setting-item">
                <span class="label">最后登录时间</span>
                <div class="line">
                    <ic-time :ago="false" :timestamp="user.key_time" />
                </div>
            </div>

            <div class="setting-item">
                <span class="label">最后访问时间</span>
                <div class="line">
                    <ic-time :ago="false" :timestamp="user.access_time" />
                </div>
            </div>

        </div>
    </div>
</setting-base>
</template>

<style scoped>
.box-avatar {
    position: relative;
}

a.resend {
    margin-left: 5px;
}

.box-avatar > .btn-upload {
    bottom: 0;
    position: absolute;
    opacity: 0.7;
    width: calc(100% - 6px);
    margin-right: 3px;
    margin-left: 3px;
}

.lbox {
    width: 85%;
}

.commentArea {
    width: 100%;
}

.setting-item {
    margin-bottom: 20px;
}

.setting-item > .label {
    font-size: 17px;
    font-weight: 500;
    display: block;
    margin-bottom: 6px;
}

.box {
    display: flex;
    flex-direction: row;
}

.box > .left {
    display: flex;
    flex: 3 0 0%;

    /* border: 1px solid #ccc; */
    flex-direction: column;
}

.box > .right {
    flex: 1 0 0%;
}

.right > .rbox {
    margin-left: 20px;
}
</style>

<script>
import api from '@/netapi.js'
import state from '@/state.js'
import SettingBase from '../base/base.vue'

export default {
    data () {
        return {
            state,
            sending: false,
            userSave: null,
            updating: false,
            avatarUploadShow: false
        }
    },
    computed: {
        'user': function () {
            if (!this.userSave) {
                this.$set(this, 'userSave', _.clone(state.user))
            }
            return this.userSave
        },
        atLeastUser: function () {
            if (this.state.user) {
                let g = this.state.user.group
                return g && g > this.state.misc.USER_GROUP.INACTIVE
            }
        },
        changed: function () {
            let data = $.objDiff(this.userSave, state.user)
            return Object.keys(data).length
        }
    },
    methods: {
        fetchData: async function () {
            // let params = this.$route.params
            // let ret = await api.topic.get({
            //     id: params.id,
            // })
        },
        resendActivationMail: async function () {
            if (this.sending) return
            this.sending = true
            let ret = await api.user.resendActivationMail()
            if (ret.code === api.retcode.SUCCESS) {
                $.message_success('激活邮件发送成功！请检查邮箱。')
            } else {
                $.message_error('发送失败，每30分钟只能发送一次。')
            }
            this.sending = false
        },
        updateInfo: async function () {
            if (this.updating) return
            if (!this.changed) return
            this.updating = true
            let data = $.objDiff(this.userSave, state.user)
            delete data['avatar'] // 头像的更新是独立的参见BUG12
            let ret = await api.user.set({ id: state.user.id }, data, 'user')
            if (ret.code === api.retcode.SUCCESS) {
                for (let [k, v] of Object.entries(data)) {
                    state.user[k] = v
                }
                this.userSave = _.clone(state.user)
                $.message_success('信息修改成功！')
            }
            this.updating = false
        }
    },
    created: async function () {
        let key = state.loadingGetKey(this.$route)
        this.state.loadingInc(this.$route, key)
        await this.fetchData()
        this.state.loadingDec(this.$route, key)
    },
    watch: {
        // 如果路由有变化，会再次执行该方法
        '$route': 'fetchData'
    },
    components: {
        SettingBase
    }
}
</script>
