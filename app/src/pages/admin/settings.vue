<template>
    <div>
        <v-card class="my-2 elevation-4" v-for="card in cards" :key="card.title" >
            <v-card-title @click="card.show = !card.show">
                <v-btn @click.once="card.show = !card.show" icon>
                    <v-icon>{{ card.show ? 'keyboard_arrow_down' : 'keyboard_arrow_up' }}</v-icon>
                </v-btn>
                {{card.title}}
            </v-card-title>
          <v-expand-transition>
            <!-- v-card-text的padding导致动画收起时卡顿，置为 0，内部用一个div来控制-->
            <v-card-text v-show="card.show" style="padding: 0">
              <div style="padding: 0 16px 16px">
                <p v-if="card.subtitle" class="">{{card.subtitle}}</p>
                <template v-if="card.tips">
                  <p v-for="t in card.tips" :key="t.text">{{t.text}} <a v-if="t.link" target="_blank" :href="t.link">链接</a></p>
                </template>

                <template v-for="f in card.fields" :key="f.key" >
                  <v-checkbox small hide-details v-if="f.type === 'checkbox' " :prepend-icon="f.icon" v-model="settings[f.key]" :label="f.label" ></v-checkbox>
                  <v-textarea outlined v-else-if="f.type === 'textarea' " :prepend-icon="f.icon" v-model="settings[f.key]" :label="f.label" ></v-textarea>
                  <v-select small hide-details v-else-if="f.type === 'select' " :prepend-icon="f.icon" v-model="settings[f.key]" :items="f.items" :label="f.label" > </v-select>
                  <v-text-field v-else :prepend-icon="f.icon" v-model="settings[f.key]" :label="f.label" type="text"></v-text-field>
                </template>
                <template v-for="b in card.buttons" :key="b.label" >
                  <v-btn @click="run(b.action)" color="primary"><v-icon>{{b.icon}}</v-icon>{{b.label}}</v-btn>
                </template>

                <template v-for="g in card.groups" :key="g.label" >
                  <v-checkbox small hide-details v-model="settings[g.key]" :label="g.label"></v-checkbox>
                  <template v-if="settings[g.key]">
                    <template v-for="f in g.fields" :key="f.key" >
                      <v-textarea outlined v-if="f.type === 'textarea' " :prepend-icon="f.icon" v-model="settings[f.key]" :label="f.label" ></v-textarea>
                      <v-text-field v-else :prepend-icon="f.icon" v-model="settings[f.key]" :label="f.label" type="text"></v-text-field>
                    </template>
                  </template>
                </template>

                <template v-if="card.show_friends">
                  <v-row v-for="(friend, idx) in settings.FRIENDS" :key="'friend-'+friend.href">
                    <v-col class='py-0' cols=3>
                      <v-text-field flat small hide-details single-line v-model="friend.text" label="名称" type="text"></v-text-field>
                    </v-col>
                    <v-col class='pa-0' cols=9>
                      <v-text-field flat small hide-details single-line v-model="friend.href" label="链接" type="text"
                                    append-outer-icon="delete" @click:append-outer="settings.FRIENDS.splice(idx, 1)" ></v-text-field>
                    </v-col>
                  </v-row>
                  <v-row>
                    <v-col align="center">
                      <v-btn color="primary" @click="settings.FRIENDS.push({text:'', href: ''})"><v-icon>add</v-icon>添加</v-btn>
                    </v-col>
                  </v-row>
                </template>

                <template v-if="card.show_socials">
                  <p>所启用的社交网络将会在登录页面自动显示按钮。</p>
                  <v-combobox v-model="settings.SOCIALS" :items="sns_items" label="选择要启用的社交网络账号" hide-selected multiple small-chips>
                    <template v-slot:selection="{ attrs, item, parent, selected }">
                      <v-chip v-bind="attrs" color="green lighten-3" :input-value="selected" label small >
                        <span class="pr-2"> {{ item.text }} </span>
                        <v-icon small @click="parent.selectItem(item)" >close</v-icon>
                      </v-chip>
                    </template>
                  </v-combobox>
                  <v-row v-for="s in settings.SOCIALS" :key="'social-'+s.value" >
                    <v-col class='py-0' cols=12 sm=2>
                      <v-subheader class="px-0 pt-4" :class="$vuetify.breakpoint.smAndUp?'float-right':''">
                        {{s.text}}  (<a @click="show_sns_config(s)">说明</a>)
                      </v-subheader>
                    </v-col>
                    <v-col class='py-0' cols=12 sm=3>
                      <v-text-field small hide-details single-line v-model="settings['SOCIAL_AUTH_'+s.value.toUpperCase()+'_KEY']" label="Key" type="text"></v-text-field>
                    </v-col>
                    <v-col class='py-0' cols=12 sm=7>
                      <v-text-field small hide-details single-line v-model="settings['SOCIAL_AUTH_'+s.value.toUpperCase()+'_SECRET']" label="Secret" type="text"></v-text-field>
                    </v-col>
                  </v-row>
                </template>

                <template v-if="card.show_ssl">
                  <ssl-manager />
                </template>
              </div>
            </v-card-text>
          </v-expand-transition>
        </v-card>

        <br/>
        <div class="text-center">
            <v-btn color="primary" @click="save_settings">保存</v-btn>
        </div>
    </div>
</template>

<script>
import SSLManager from "~/components/SSLManager.vue";
export default {
    components: {
        "ssl-manager": SSLManager,
    },
    created() {
        this.$backend("/admin/settings").then(rsp => {
            this.sns_items = rsp.sns;
            this.settings = rsp.settings;
            this.site_url = rsp.site_url;

            var m = {}
            rsp.sns.forEach(function(ele){
                m[ele.value] = ele;
            });
            this.settings.SOCIALS.forEach(function(ele){
                ele.help = false;
                ele.link = m[ele.value].link;
            })
        });
    },
    data: () => ({
        combo_input: "",
        sns: {},
        sns_items: [],
        settings: { },
        site_url: "",

        cards: [
            {
            show: false,
            title: "基础信息",
            fields: [
                { icon: "home", key: "site_title", label: "网站标题", },
                { icon: "mdi-copyright", key: "HEADER", label: "网站公告", type: 'textarea' },
                { icon: "mdi-copyright", key: "FOOTER", label: "网站脚注", type: 'textarea' },
            ],
            groups: [
            {
                key: "INVITE_MODE",
                label: "开启私人图书馆模式",
                fields: [
                    { icon: "lock", key: "INVITE_CODE", label: "访问码" },
                    { icon: "person", key: "INVITE_MESSAGE", type: 'textarea', label: "提示语" },
                ],
            },
            ],
        },
        {
            show: false,
            title: "用户设置",
            fields: [
                { icon: "", key: "ALLOW_GUEST_READ", label: "允许访客在线阅读（无需注册和登录）", type: 'checkbox' },
                { icon: "", key: "ALLOW_GUEST_DOWNLOAD", label: "允许任意下载（访客无需注册和登录）", type: 'checkbox' },
                { icon: "", key: "ALLOW_GUEST_PUSH", label: "允许任意推送Kindle（访客无需注册和登录）", type: 'checkbox' },
            ],
            groups: [
            {
                key: "ALLOW_REGISTER",
                label: "允许访客以邮箱注册账号",
                fields: [
                    { icon: "info", key: "SIGNUP_MAIL_TITLE", label: "激活邮件标题" },
                    { icon: "info", key: "SIGNUP_MAIL_CONTENT", label: "激活邮件正文", type: 'textarea' },
                    { icon: "info", key: "RESET_MAIL_TITLE", label: "重置密码邮件标题" },
                    { icon: "info", key: "RESET_MAIL_CONTENT", label: "重置密码邮件正文", type: 'textarea' },
                ],
            },
            ],
        },

        {
            show: false,
            title: '社交网络登录',
            fields: [ ],
            show_socials: true,
        },
        {
            show: false,
            title: "邮件服务",
            subtitle: '邮箱注册、推送Kindle依赖此配置(SMTP服务器地址可带端口，或者不带端口，默认为465号)',
            fields: [
                { icon: "email", key: "smtp_server", label: "SMTP服务器（例如 smtp-mail.outlook.com:587）" },
                { icon: "person", key: "smtp_username", label: "SMTP用户名（例如 user@gmail.com）" },
                { icon: "lock", key: "smtp_password", label: "SMTP密码" },
                { icon: "info", key: "smtp_encryption", label: "SMTP安全性", type: 'select',
                    items: [{text: "SSL", value: "SSL"}, {text: "TLS(多数邮箱为此选项)", value: "TLS"} ]
                },
            ],
            buttons: [
                { icon: "email", label: "测试邮件", action: "test_email" },
            ],
        },
        {
            show: false,
            title: "书籍标签分类",
            subtitle: '配置「分类导航」页面里预设的分类。添加书籍时，若书名或者作者名称出现以下分类，则自动添加对应的标签。',
            fields: [
                { icon: "person", key: "BOOK_NAV", type: 'textarea', label: "分类" },
            ],
        },
        {
            show: false,
            title: '友情链接',
            fields: [ ],
            show_friends: true,
        },

        {
            show: false,
            title: "互联网书籍信息源",
            fields: [
                { icon: "", key: "auto_fill_meta", label: "自动从互联网拉取新书的书籍信息", type: 'checkbox' },
                { icon: "info", key: "douban_baseurl", label: "豆瓣插件API地址(例如 http://10.0.0.1:8080 )" },
                { icon: "info", key: "douban_max_count", label: "豆瓣插件API查询结果数量" },
            ],
            tips: [
                {
                    text: "若需要启用豆瓣插件，请参阅安装文档的说明。若出现失败，可尝试更换镜像，例如 talebook/douban-api-rs ",
                    link: "https://github.com/talebook/talebook/blob/master/document/README.zh_CN.md#%E5%A6%82%E6%9E%9C%E9%85%8D%E7%BD%AE%E8%B1%86%E7%93%A3%E6%8F%92%E4%BB%B6",
                }
            ],
        },

        {
            show: false,
            title: "高级配置项",
            fields: [
                { icon: "home", key: "static_host", label: "CDN域名" },
                // 后续可以修改为choice下拉框选项
                { icon: "info", key: "BOOK_NAMES_FORMAT", label: "目录和文件名模式", type: 'select',
                    items: [{text: "使用拼音字母目录名 (兼容性高)", value: "en"}, {text: "使用中文目录名 (UTF8编码，更美观)", value: "utf8"} ]
                },
                { icon: "info", key: "EPUB_VIEWER", label: "EPUB阅读器", type: 'select',
                    items: [{text: "Epub Reader（旧版）", value: "epubjs.html"}, {text: "Candle Reader（Beta版，支持章评功能）", value: "creader.html"} ]
                },
                { icon: "info", key: "avatar_service", label: "可使用www.gravatar.com或cravatar.cn头像服务" },
                { icon: "info", key: "MAX_UPLOAD_SIZE", label: "文件上传字节数限制(例如100MB或100KB）" },
                { icon: "lock", key: "cookie_secret", label: "COOKIE随机密钥" },
                { icon: "info", key: "scan_upload_path", label: "批量导入扫描目录" },
                { icon: "info", key: "push_title", label: "邮件推送的标题" },
                { icon: "info", key: "push_content", label: "邮件推送的内容" },
                { icon: "info", key: "convert_timeout", label: "书籍转换格式的最大超时时间（秒）" },
                { icon: "", key: "autoreload", label: "更新配置后自动重启服务器(首次开启需人工重启)", type: 'checkbox' },
            ],
            tips: [
                {
                    text: "若需要调整Logo，请参阅安装文档的说明。",
                    link: "https://github.com/talebook/talebook/blob/master/document/README.zh_CN.md#logo",
                }
            ],
        },

        {
            show: false,
            title: "SSL证书管理",
            fields: [],
            show_ssl: true,
        },

        ],
    }),
    methods: {
        save_settings: function() {
            this.$backend("/admin/settings", {
                method: 'POST',
                body: JSON.stringify(this.settings),
            })
            .then( rsp => {
                if ( rsp.err != 'ok' ) {
                    this.$alert('error', rsp.msg);
                } else {
                    this.$alert('success', '保存成功！可能需要5~10秒钟生效！');
                }
            });
        },
        show_sns_config: function(s) {
            var msg = `请前往${s.text}的 <a :href="${s.link}" target="_blank">配置页面</a> 获取密钥，并设置回调地址（callback URL）为
            <code>${this.site_url}/auth/complete/${s.value}.do</code>`;
            this.$alert("success", msg);
        },
        test_email: function() {
            var data = new URLSearchParams();
            data.append('smtp_server', this.settings['smtp_server']);
            data.append('smtp_username', this.settings['smtp_username']);
            data.append('smtp_password', this.settings['smtp_password']);
            data.append('smtp_encryption', this.settings['smtp_encryption']);
            this.$backend("/admin/testmail", {
                method: 'POST',
                body: data,
            }).then( rsp => {
                if ( rsp.err != 'ok' ) {
                    this.$alert('error', rsp.msg);
                } else {
                    this.$alert('success', rsp.msg);
                }
            });
        },
        run: function(func) {
            this[func]();
        },
    },
  }
</script>

<style>
.cursor-pointer {
    cursor: pointer;
}
</style>

