<template>
  <div style="max-width: 800px;">
    <amplify-authenticator>
      <v-text-field
        label="コメント"
        placeholder="ここにコメントを書きましょう"
        outlined
        class="mx-auto"
        append-icon="mdi-check-bold"
        style="max-width: 100%; box-sizing: border-box;"
        v-model="form.comment"
        @keydown="onEnter"
        @click:append="createChat"
      ></v-text-field>
    </amplify-authenticator>

    <v-card v-for="(item, index) in items" :key="index" tile>
      <v-list-item two-line>
        <v-list-item-content>
          <v-list-item-title>{{ item.comment }}</v-list-item-title>
          <v-list-item-subtitle>by: {{ item.owner }}</v-list-item-subtitle>
        </v-list-item-content>
      </v-list-item>
    </v-card>
    <v-card>
      <amplify-sign-out v-if="logoutBtn"></amplify-sign-out>
    </v-card>
  </div>
</template>
<script>
import { API } from "aws-amplify"; // Amplifyライブラリを読み込み
import { createChat } from "~/graphql/mutations"; // GraphQL Mutation（データをエンドポイントに送信する構文?）
import { listChats } from "~/graphql/queries"; // GraphQL Query（データを読み込む構文？）
import { onAuthUIStateChange } from "@aws-amplify/ui-components";
import { onCreateChat } from "~/graphql/subscriptions";

export default {
  data() {
    return {
      form: {
        comment: ""
      },
      items: [],
      logoutBtn: false
    };
  },
  created() {
    this.getChatList();
    this.subscribe();
    onAuthUIStateChange((authState, authData) => {
      // ログインステータスが変化したとき
      if (authState === "signedin") {
        // ログインした場合
        this.getChatList(); // 一覧呼び出し
        this.subscribe(); // GraphQL Subscription
        this.logoutBtn = true;
      } else {
        this.items = []; // ログアウトしたときなどは一覧を削除
        this.logoutBtn = false;
      }
    });
  },
  methods: {
    async createChat() {
      // コメントを送信する
      const comment = this.form.comment; // コメント入力値を取得
      if (!comment) return; // 空のときは処理しない
      const chat = { comment }; // 送信用のJSONを作成
      // 送信処理
      await API.graphql({
        query: createChat, // GraphQL Mutation
        variables: { input: chat } // 送信データ
      });
      this.form.comment = ""; // 送信後にテキストフィールドを空に。
    },
    onEnter(event) {
      // ここはおまけ。（Enterを押したときもコメントを送信したかったので記述）
      if (event.keyCode !== 13) return;
      console.log("save");
      this.createChat();
    },
    async getChatList() {
      // コメント一覧を取得
      const chatList = await API.graphql({
        query: listChats // GraphQL Query
      });
      this.items = chatList.data.listChats.items; // 読み込みしたデータを一覧に表示
    },
    subscribe() {
      API.graphql({ query: onCreateChat }).subscribe({
        next: eventData => {
          // コメントが送信されて追加されたとき、送信内容を一覧に追加
          const chat = eventData.value.data.onCreateChat; // データを読み込み
          if (this.items.some(item => item.comment === chat.comment)) return; // すでに表示されているデータは無視
          this.items = [...this.items, chat]; // 新しいデータを追加
        }
      });
    }
  }
};
</script>
