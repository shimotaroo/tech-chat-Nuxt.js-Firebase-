<template>
  <div class="relative">
    <div class="px-4 pb-32">
      <!-- チャットの吹き出し部分 -->
      <div v-for="chat in chats" :key="chat.id">
        <!-- 自分の発言の場合 -->
        <template v-if="isMyChat(chat.userId)">
          <div class="flex my-4 justify-end">
            <div class="bg-iceblue rounded-tr-md rounded-l-md p-4 mt-1 w-72">
              <pre class="whitespace-pre-wrap text-sm">{{ chat.body }}</pre>
            </div>
          </div>
        </template>
        <!-- 相手の発言の場合 -->
        <template v-else>
          <div class="flex my-4">
            <div class="mr-2 mt-2 min-w-3">
              <img :src="chat.iconImageUrl" class="rounded-full h-10 w-10" />
            </div>
            <div class="w-72">
              <span class="text-xs">{{ chat.name }}</span>
              <div class="bg-gray-300 rounded-bl-md rounded-r-md p-4 ">
                <pre class="whitespace-pre-wrap text-sm">{{ chat.body }}</pre>
              </div>
            </div>
          </div>
        </template>
      </div>
    </div>
    <form
      @submit.prevent="onSubmit"
      class="fixed bottom-0 bg-white w-full max-w-sm flex py-4 border-t border-gray-300"
    >
      <textarea
        v-model="form.message.val"
        placeholder="発言してみよう"
        class="block appearance-none w-full ml-2 py-3 px-4 rounded-lg border border-gray-400 text-darkGray focus:outline-none focus:bg-white overflow-hidden bg-blue-100"
        name="form.body"
      />
      <button
        :disable="isValidateError"
        :class="{ 'text-blue': !isValidateError }"
        class="w-2/12 flex items-center justify-center text-gray font-semibold"
      >
        送信
      </button>
    </form>
  </div>
</template>

<script>
import { mapGetters, mapMutations } from 'vuex'

export default {
  middleware: ['checkAuth'],
  data () {
    return {
      form: {
        message: {
          val: null
        }
      },
      unsubscribe: null
    }
  },

  computed: {
    ...mapGetters('chats', ['chats']),
    isValidateError () {
      return !this.form.message.val
    }
  },

  async asyncData ({ store, params }) {
    const roomId = params.id
    const unsubscribe = await store.dispatch('chats/subscribe', { roomId })
    return {
      unsubscribe
    }
  },

  destroyed () {
    this.$store.dispatch('chats/clear')
    if (this.unsubscribe) {
      this.unsubscribe()
    }
  },

  methods: {
    ...mapMutations('alert', ['setMessage']),
    isMyChat (userId) {
      const { uid } = this.$fireAuth.currentUser
      return userId === uid
    },
    // データ登録
    async onSubmit () {
      if (this.isValidateError) {
        return
      }

      const user = await this.$user()

      // 未ログインの場合
      if (!user) {
        this.$router.push('/login')
      }

      // _からファイル名やディレクトリ名が始まる場合は、実際の画面を表示するときの path は http://3000/rooms/ルームID
      // 今回で言うとpages/rooms/_id.vue
      // 詳細：https://router.vuejs.org/ja/api/#%E3%83%AB%E3%83%BC%E3%83%88%E3%82%AA%E3%83%96%E3%82%B8%E3%82%A7%E3%82%AF%E3%83%88%E3%83%97%E3%83%AD%E3%83%91%E3%83%86%E3%82%A3
      const roomId = this.$route.params.id

      // 登録データを準備
      const chat = {
        userId: user.uid,
        name: user.name,
        iconImageUrl: user.iconImageUrl,
        body: this.form.message.val,
        createdAt: this.$firebase.firestore.FieldValue.serverTimestamp()
      }

      try {
        // Firestoreに登録
        // set ではなく add とすることで Firestore で UID を自動生成してデータを追加できる
        await this.$firestore
          .collection('rooms')
          .doc(roomId)
          .collection('chats')
          .add(chat)
        this.resetForm()
        this.scrollBottom()
      } catch (e) {
        this.setMessage({ message: '登録に失敗しました。' })
      }
    },

    scrollBottom () {
      const element = document.documentElement
      const bottom = element.scrollHeight - element.clientHeight
      window.scroll(0, bottom)
    },

    resetForm () {
      this.form.message.val = null
    }
  }
}
</script>
