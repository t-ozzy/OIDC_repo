# [初学者向け] NextAuthを通してのOAuth,OIDC入門

## 対象
- 基本的なhttpのステータスを理解している人
- Next.jsの大方の仕組み(ディレクトリ階層がページのパスになる、``/api``以下はNextサーバー側の処理をかけるetc)を理解できている人
- けど、NextAuth触ったこと無いし、OAuth/OIDCのフローが分からない

## フロー
ブラウザから見ると、大きく分けて２つのフローに分けられます
- singInボタンを押すと、プロパイダーが提供する認証許可画面が帰ってくる(responseはhtml)
- 認証許可画面で「許可する」ボタンを押すと、"アクセストークンやIDトークンの情報を含んだJWT"に紐づくセッションキーがSet-Cookieで帰ってくる(responseはjson？？)

以下で各フローの詳細を説明します
今回はProviderをGitHubとします

### signInボタンから認証許可画面までのフロー
1. singInボタンを押し、以下のリクエストを送る
ブラウザ　-``GET /api/auth/signin/github``->　Next サーバー
2. Nextサーバーは 認可サーバー(GitHub)の認可エンドポイントにリダイレクトするレスポンスを返す
ブラウザ　<-``302 Found
Location: https://github.com/login/oauth/authorize?
client_id=xxxxx&
redirect_uri=https://yourapp.com/api/auth/callback/github&
response_type=code``-　Next サーバー
3. ブラウザから認可サーバーへリダイレクトする
ブラウザ　-`GET https://github.com/login/oauth/authorize?
client_id=xxxxx&
redirect_uri=https://yourapp.com/api/auth/callback/github&
response_type=code`->　認可サーバー
4. 認可サーバーから認証画面が帰ってくる
ブラウザ　<-``${認証画面のHTML}``-　認可サーバー


### 「許可する」ボタンを押すと、"トークン情報を含んだJWT"が帰ってくるフロー
1. ユーザーが認可ページで「許可」を押す
このときのリクエストは、HTMLのformタグ内でボタンをsubmit()したときのPOSTリクエストです

2. 認可サーバーがNextサーバーのredirect_uriにリダイレクトするレスポンスを返す
ブラウザ　<-``302 Found
Location: https://yourapp.com/api/auth/callback/github?code=abc123``-　認可サーバー
3. ブラウザからNextサーバーへリダイレクトする
ただし、このパスは``/api``とある通り、ブラウザではなく、Nextサーバーで実行されるwebAPIが動きます
ブラウザ　-``GET /api/auth/callback/github?code=abc123``->　Nextサーバー
4. ``/api/auth/callback/github``API内で認可サーバーにリクエストを送る
Nextサーバー　-`POST https://github.com/login/oauth/access_token
body: {
client_id=xxxxx
client_secret=yyyyy
code=abc123} `->　認可サーバー
5. 認可サーバーが Next サーバーにアクセストークンを返す
Next サーバー　<-`200 OK body: {
  "access_token": "abcdef...",
  "token_type": "bearer",
  "scope": "user:email"
} `-　認可サーバー
6. Next サーバーが各ブラウザ(ユーザー)を識別するための Cookie を返す
Nextはそのままjwtを返しません
jwtは自身が保持している情報が改ざんされていないかを保証しますが、jwtを自身を奪われてしまえば、中身を見られてしまいます
そのため、Nextサーバー内で暗号化し、トークンが奪われてしまっても、中身が確認できないようにします(ただ、なりすましはできてしまう)
暗号化したトークンをセッションキーとしてブラウザに返します
ブラウザ <-`Set-cookie: a1b2c3de...`- Next サーバー


## 情報の取得
先程書いたように、OAuth/OIDC認証が成功した場合、jwtに情報が取得されます。
その情報にどうやってアクセスするのかというと、useSession()というNextAuthが提供してくれる関数で取得できます

jwtを生成するときに、jwtに含めたい情報のカスタマイズ、
useSession()を読んだ際に、取得したい情報のカスタマイズをしたいときは、
NextAuth()に渡す``callbacks: {jwt, session}``関数でそれぞれ編集できます

```ts
const handler = NextAuth({
  providers: [
    GitHubProvider({
      clientId: "your-clientId",
      clientSecret: "your-clientSecret",
    }),
  ],
  callbacks: {
  　async jwt({token, account}) {
      //認証が成功し、jwtが生成された際、追加したい情報をカスタマイズできる
      if (account) {
        token.accessToken = account.access_token;
      }
      return token;
    },
    async session({ session, token }) {
      //この中で取得できる情報をカスタマイズできる
      if (token) {
        session.accessToken = token.accessToken as string | undefined;
      }
      return session;
    },
  },
});

```

## コードチャレンジ、pkce
