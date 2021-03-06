<template>
<loading v-if="pageLoading"/>
<div v-else class="ic-container">
    <div class="edit-page-title">
        <div v-title>{{ is_edit ? '编辑主题' : '新建主题' }} - {{state.config.title}}</div>
        <h3 class="" v-if="!is_edit">新建主题</h3>
        <h3 class="" v-else>编辑主题<span v-if="asAdmin"> - 管理员模式</span></h3>
        <button class="ic-btn primary right-top-btn" type="primary" :loading="loading" @click="send">{{postButtonText}}</button>
    </div>

    <form class="ic-form" id="form_topic" method="POST" @submit.prevent="send">
        <check-row :results="formErrors.title" :multi="true">
            <input type="text" name="title" v-model="topicInfo.title" :placeholder="`这里填写标题，${state.misc.BACKEND_CONFIG.TOPIC_TITLE_LENGTH_MIN} - ${state.misc.BACKEND_CONFIG.TOPIC_TITLE_LENGTH_MAX} 字`">
        </check-row>
        <check-row :results="formErrors.board" :multi="true" v-if="(!is_edit) || asAdmin">
            <multiselect v-model="topicInfo.board_id" :allow-empty="false" :options="boardList" :custom-label="getSelectOptionName" placeholder="选择一个板块" style="z-index: 2" open-direction="bottom"></multiselect>
        </check-row>
        <check-row :results="formErrors.content" :multi="true">
            <markdown-editor ref="editor" :configs="mdeConfig" v-model="topicInfo.content" rows="15" autofocus></markdown-editor>
        </check-row>
        <div class="ic-form-row">
            <button class="ic-btn primary" style="float: right" type="primary" :loading="loading">{{postButtonText}}</button>
        </div>
    </form>
</div>
</template>

<style lang="scss" scoped>
#form_topic input[name='title'] {
    padding: 6px 12px;
    width: 100%;
    font-weight: bolder;
    border-radius: 5px;
    box-shadow: none;
    border: 1px solid #e8e8e8;
    outline: none;
}

.error {
    #form_topic input[name='title'] {
        border: 1px solid $danger;
    }
}

.edit-page-title {
    display: flex;
    padding: 20px 0;
    justify-content: space-between;
    align-items: center;
}

.ic-form-row {
    margin-bottom: 10px;
}

.form-select-row {
    /* 此举是为了避免 ic-form 对 input 的样式覆盖。
        不然select会出现外框套内框这样的事情 */
    margin-bottom: 10px;
}

.el-select > .el-input > input[readonly] {
    background-color: #fff;
    color: #1f2d3d;
}
</style>

<style>
div.markdown-editor > div.editor-toolbar {
    border-color: #ddd !important;
}

/* 这部分内容是动态生成的 因此写在scope中无效 */
.ic-form > .ic-form-row.error > div.markdown-editor > div.editor-toolbar {
    opacity: 1; /* 上下颜色一致 */
    border-color: #FF6060 !important;
}

.ic-form > .ic-form-row.error > .markdown-editor > .cm-s-easymde {
    border-left-color: #FF6060 !important;
    border-right-color: #FF6060 !important;
    border-bottom-color: #FF6060 !important;
}

.editor-preview img {
    width: 100%;
}

.multiselect__tags {
    border: 1px solid #ddd !important;
}
</style>

<script>
import Prism from 'prismjs'
import Multiselect from 'vue-multiselect'
import 'vue-multiselect/dist/vue-multiselect.min.css'
import markdownEditor from '@/components/misc/markdown-editor.vue'
import api from '@/netapi.js'
import state from '@/state.js'
import nprogress from 'nprogress/nprogress.js'
import * as qiniu from 'qiniu-js'
import Objectid from 'objectid-js'
// import marked from '@/md.js'

export default {
    data () {
        return {
            state,
            asAdmin: false,
            pageLoading: true,
            loading: false,
            boardList: [],
            boardInfo: {},
            save: {},

            topicInfo: {
                title: '',
                board: null,
                board_id: null,
                content: ''
            },

            formErrors: {
                title: [],
                content: []
            },

            mdeConfig: {
                spellChecker: false,
                autoDownloadFontAwesome: false,
                placeholder: '这里填写内容，支持 Markdown 格式。\n支持图片上传（GIF除外），可通过拖拽或粘贴进行上传，大小限制5MB。',
                autosave: {
                    enabled: false,
                    uniqueId: 'topic-post-content'
                },
                renderingConfig: {
                    singleLineBreaks: false,
                    codeSyntaxHighlighting: false
                },
                previewRender: function (plainText, preview) { // Async method
                    setTimeout(function () {
                        preview.innerHTML = this.parent.markdown(plainText)
                        Prism.highlightAll()
                    }.bind(this), 1)
                    return 'Loading...'
                }
            },

            topicState: state.misc.POST_STATE.NORMAL
        }
    },
    computed: {
        is_edit () {
            return this.$route.name === 'forum_topic_edit'
        },
        postButtonText: function () {
            return this.loading ? '请等待'
                : (this.is_edit ? '编辑' : '发布')
        }
    },
    methods: {
        getSelectOptionName (id) {
            let { name, brief } = this.boardInfo[id]
            return `${name} — [${brief}]`
        },
        send: async function (e) {
            let ret
            let successText
            let failedText

            if (!this.topicInfo.board_id) {
                $.message_error('没有选择发布的板块，如果没有板块，请先在管理界面创建。')
                return
            }

            let topicId
            if (this.loading) return
            this.loading = true

            let topicInfo = $.objDiff({
                'title': this.topicInfo.title,
                'board_id': this.topicInfo.board_id,
                'content': this.topicInfo.content,
                'returning': true
            }, this.save)

            if (this.is_edit) {
                if (this.asAdmin) {
                    ret = await api.topic.set({ id: this.topicInfo.id }, topicInfo, 'superuser')
                } else {
                    ret = await api.topic.set({ id: this.topicInfo.id }, topicInfo, 'user')
                }
                successText = '编辑成功！已自动跳转至文章页面。'
                failedText = ret.msg || '编辑失败！'
                topicId = this.topicInfo.id
            } else {
                ret = await api.topic.new(topicInfo, 'user')
                successText = '发表成功！已自动跳转至文章页面。'
                failedText = ret.msg || '新建失败！'
                topicId = ret.data.id
            }

            if (ret.code === 0) {
                localStorage.setItem('topic-post-cache-clear', 1)
                this.$router.push({ name: 'forum_topic', params: { id: topicId } })
                $.message_success(successText)
            } else if (ret.code === api.retcode.INVALID_ROLE) {
                $.message_error('抱歉，您的账户为未激活账户，无法发表主题，请检查邮件。若未收到，请在设置界面重新发送激活邮件。')
                this.loading = false
            } else {
                if (ret.code === api.retcode.FAILED) {
                    this.formErrors = ret.data
                }
                $.message_error(failedText)
                // 注意：发布成功会跳转，故不做复位，失败则复位
                this.loading = false
            }
        },
        fetchData: async function () {
            this.pageLoading = true
            let params = this.$route.params

            this.asAdmin = params.asAdmin
            if (!state.user) {
                $.message_error('抱歉，无权访问此页面，请返回')
                return
            }

            let ret = await api.board.list()
            if (ret.code) {
                $.message_by_code(ret.code)
                return
            }
            let boardList = ret.data.items

            if (this.is_edit) {
                let ret = await api.topic.get({
                    id: params.id,
                    loadfk: { user_id: null, board_id: null }
                })
                if (ret.code) {
                    $.message_error('抱歉，发生了错误')
                    return
                }

                /* eslint-disable no-control-regex */
                ret.data.content = ret.data.content.replace(/\x01(.+?)\x01/g, '@$1')
                this.topicInfo = ret.data
                this.save = _.clone(ret.data)
                this.save.board_id = this.save.board_id.id
            }

            this.boardList = []
            for (let board of boardList) {
                this.boardInfo[board.id] = board
                this.boardList.push(board.id)
            }

            if (boardList.length) {
                if (!this.is_edit) {
                    // 若是新建，给一个默认板块
                    this.topicInfo.board_id = boardList[0].id
                    // 若指定了板块id，那么按设置来
                    if (params.board_id) {
                        for (let i of boardList) {
                            if (i.id === params.board_id) {
                                this.topicInfo.board_id = i.id
                                break
                            }
                        }
                    }
                } else {
                    // 若不是新建，那么按文章的板块来
                    for (let i of boardList) {
                        if (i.id === this.topicInfo.board_id.id) {
                            this.topicInfo.board_id = i.id
                            break
                        }
                    }
                }
            }

            this.pageLoading = false
        }
    },
    beforeRouteEnter (to, from, next) {
        if (!state.user) {
            state.loading = 0
            nprogress.done()
            $.message_error('在登录后才能发帖。请登录账号，如果没有账号，先注册一个。')
            return next('/')
        }
        next()
    },
    mounted: async function () {
        // if (localStorage.getItem('topic-post-cache-clear')) {
        //     // 我不知道为什么，在地址跳转前进行 storage 的清除工作，
        //     // 并不会实质上起效，因此这是一个替代手段，效果比较理想。
        //     localStorage.removeItem('topic-post-title')
        //     localStorage.removeItem('smde_topic-post-content')
        //     localStorage.removeItem('topic-post-cache-clear')
        // }

        if (!this.is_edit) {
            // this.title = localStorage.getItem('topic-post-title') || ''
        }
    },
    created: async function () {
        await this.fetchData()

        let func = async () => {
            let editor = this.$refs.editor
            if (editor) {
                let uploadImage = async function (editor, fileList) {
                    let theFile = null

                    if (fileList.length === 0) return
                    for (let i of fileList) {
                        if (i.type.indexOf('image') !== -1) {
                            theFile = i
                            break
                        }
                    }
                    if (!theFile) return false
                    let token = await $.asyncGetUploadToken()

                    let placeholder = `![Uploading ${theFile['name']} - ${(new Objectid()).toString()} ...]()`
                    editor.replaceRange(placeholder, {
                        line: editor.getCursor().line,
                        ch: editor.getCursor().ch
                    })
                    let ob = qiniu.upload(theFile, null, token, null)

                    ob.subscribe({
                        complete: (ret) => {
                            // 注意，这里的res是本地那个callback的结果，七牛直接转发过来了
                            // console.log('done', ret)
                            if (ret.code === api.retcode.SUCCESS) {
                                // let url = `${config.qiniu.host}/${ret.data}` // -${config.qiniu.suffix}
                                let url = `${state.misc.BACKEND_CONFIG.UPLOAD_STATIC_HOST}/${ret.data}`
                                let suffix = state.misc.BACKEND_CONFIG.UPLOAD_QINIU_IMAGE_STYLE_TOPIC
                                if (suffix) url += `-${suffix}`
                                let newTxt = `![](${url})`
                                let offset = newTxt.length - placeholder.length
                                let cur = editor.getCursor()
                                editor.setValue(editor.getValue().replace(placeholder, newTxt + '\n'))
                                editor.setCursor(cur.line, cur.ch + offset)
                            }
                        }
                    })
                }

                let cm = editor.simplemde.codemirror
                cm.on('drop', async (editor, e) => {
                    await uploadImage(editor, e.dataTransfer.files)
                })
                cm.on('paste', async (editor, e) => {
                    await uploadImage(editor, e.clipboardData.files)
                    return false
                })
            } else {
                setTimeout(func, 500)
            }
        }
        setTimeout(func, 500)
    },
    watch: {
        'topicInfo.title': _.debounce(function (val, oldVal) {
            localStorage.setItem('topic-post-title', val)
        }, 5000)
    },
    components: {
        Multiselect,
        markdownEditor
    }
}
</script>
