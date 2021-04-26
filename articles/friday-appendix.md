---
title: "週刊誌の動画特典ページをNext.js + Vercelで構築した話"
emoji: "📹"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "nextjs", "vercel"]
published: false
---

## はじめに
KODANSHAtechという会社でFRIDAYの開発案件に携わっています。
FRIDAYは講談社から発行されている写真週刊誌です。スクープ記事が注目されがちですがグラビアも多くやっており、撮影時の動画をウェブで提供しています。本記事では、FRIDAYにおける動画特典ページをNext.js ＋ Vercelでリプレースした事例を紹介します。

![](https://storage.googleapis.com/zenn-user-upload/xwktwmtbrwjjhv9jzsq59p3o6yk7)

上記の画像はユーザーが動画を見るまでのフローを示したものです。ユーザーはFRIDAY紙面に掲載されたQRコードを読み込み、遷移先のページでパスワードを入力します。入力すると、動画を視聴できるようになります。

## 移行の経緯・技術選定
FRIDAYには古くから存在する月額課金のサービスが存在し、そのサービスに相乗りする形で紙面の動画特典ページを運用していました。しかし、[サブスクリプションのサービスにリプレース](https://friday.gold)したタイミングで、動画特典ページを別アプリとして切り出すことに決めました。

動画特典ページは不定期で必要になり、動画ごとに1ページを生成するというものです。そのため、更新頻度は低く、ほとんど静的なページで構成できることが特徴です。

FRIDAYではNext.js + TypeScriptのアプリケーションを複数運用しており、とくに理由がなければこのスタックを採用しています。上記の特徴からもNext.jsと相性がよいことから、本アプリケーションでもNext.jsを採用しました。

技術選定でもう1つの大きなテーマがインフラです。チーム内ではAWSを使うことが多かったのですが、今回の開発では開発期間が十分でなく、仕様もシンプルです。その前提を踏まえるとNext.jsの開発元であるVercelのサービスの利用が適しているように思えました。
Vercelの課金形態ではProプランに収めないと、（高額と噂の）Enterpriseプランが必要になりますが、事前にProプランの制限で十分ということは確認しました。
そこでテックリードに確認したところ「Vercelでいってみよう」となったので、Vercelを利用することになりました。本当に即日で採用に至りました。

## 基本的な仕組み

### ページの生成
動画特典ページを発行するたびに、新たなページを生成する必要があります。こういった要件に対してはNext.jsの[Static Generation](https://nextjs.org/docs/basic-features/data-fetching#getstaticprops-static-generation)が有効です。
Next.jsではpages/ディレクトリにおいたコンポーネントがページとして認識されます。このファイル内で`getStaticProps`, `getStaticPaths`をexportすることで小さな静的サイトジェネレーターのような振る舞いを持たせられます。

```ts:pages/[slug].tsx
import { GetStaticPaths, GetStaticProps, NextPage } from 'next';

const pageProps = {
  slug1: {
    title: ""
  },
  slug2: {
    title: ""
  },
}

// 略
const SlugPage = () => {}

export default SlugPage

export const getStaticProps: GetStaticProps = async (context) => {
  const slug = context.params.slug as string;
  const props = pageProps[slug].props;
  return { props };
};

export const getStaticPaths: GetStaticPaths = async () => {
  return {
    paths: Object.keys(pageProps).map((key) => {
      return { params: { slug: key } };
    }),
    fallback: false,
  };
};
```

### パスワードの処理
各ページから入力されたパスワードをAPI Routesで処理します。値が正しい場合、動画のURLをレスポンスに乗せる方針を採用しました。Next.jsのAPI Routesは複雑な処理を行うには向いていないと考えていますが、今回のように1エンドポイントであればNext.jsで処理を行ったほうがシンプルだと考えました。
次のコードはパスワードの処理を簡略化したものです。（エラー処理等は省略しています）

```ts:pages/api/hoge.ts
import { NextApiRequest, NextApiResponse } from 'next';

export default (req: NextApiRequest, res: NextApiResponse): void => {
  const { slug, value } = JSON.parse(req.body);
  const url = getVideoUrl(slug, value)
  res.statusCode = 200;
  res.json({ url });
};
```

### データの定義方法
ページで扱うデータは2種類に分けられます。タイトルやクレジットなどの情報はページコンポーネントから、パスワードと実際の動画URLはAPI Routesで利用します。
後者の情報はクライアントのコードに露出してはいけませんが、データを定義する段階ではまとまった場所に置いておきたいです。
そこで、次のようなデータを定義するファイルを生成しました。

```ts:data.ts
type PageData = {
  props {
    title: string;
    poster: string;
  };
  secret: {
    password: string;
    url: string;
  }
}

const data: Record<string, PageData> = {
  slug1: {
    props: {
      title: "",
      poster: ""
    },
    secret: {
      password: "",
      url: ""
    }
  },
  slug2: {
    props: {
      title: "",
      poster: ""
    },
    secret: {
      password: "",
      url: ""
    }
  },
}

export default data
```

オブジェクトのキーはページのパスを示し、propsはgetStaticPropsで利用する値、secretはAPI Routesで利用する値にしています。
Page ComponentとAPI Routesでそれぞれこのファイルを読み込んでいます。

## 動画の設置
FRIDAYでは画像・動画はCloudinaryにアップロードしたものを利用しています。本アプリケーションでもその方針に従いCloudinaryに動画をアップロードし、HLS形式で配信を行っています。
[HLSはモダンブラウザではSafariにしか対応していない](https://caniuse.com/?search=hls)ので、他ブラウザで再生するには何らかの対応が必要です。video.jsやhls.jsなどで対応が行えますが、本件ではUIを持たないhls.jsを採用しています。

また、動画プレイヤーで見落としがちなposter画像についても必須項目としています。poster属性を指定することで動画をダウンロードする前でも動画が読み込まれているかのように見せることができます。
[web.devの記事](https://web.dev/video-and-source-tags/#include-a-poster-image)がよく書かれています。動画を扱う際には、ひととおり目を通すと良いでしょう。

## 分析の方針
せっかく内製にするのであれば、PV・UUだけでなく、もっと細かい値をとってコンテンツ作りにフィードバックしたいと考えていました。
動画コンテンツという軸でいえば、YouTubeのアナリティクスが挙げられます。YouTubeのアナリティクスでは視聴回数だけでなく平均視聴時間や視聴者維持率といったものが表示されます。チャンネルのオーナーはこのようなデータを見ながら動画の改善を行えるようになっています。
また、動画コンテンツ以外にも、ユーザーは迷わずパスワードを入力してくれているのか？、間違えずに入力してくれているのか？といった疑問を解消する手段があると良いでしょう。

そういった考えをもとに実装を行いました。
動画のトラッキングは[HTMLMediaElement](https://developer.mozilla.org/ja/docs/Web/API/HTMLMediaElement)のイベントを使うことで実現します。今回はNext.jsを使っているのでCustom Hooksにまとめて、いい感じにデータを取れるようにしています。
かなり簡略化していますが、次のようなCustom Hooksを利用しています。結構便利です。

```ts:useVideoAnalytics.ts
export const useAnalytics = (
  ref: MutableRefObject<HTMLVideoElement>,
  deps: ReadonlyArray<any>
): void => {
  const track = useTracker();
  useEffect(() => {
    const player = ref.current;
    if (!player) return;
    let maxPercent = 0;
    const sentEvents = { start: false, progress50: false, complete: false };

    const play = () => {
      if (sentEvents.start) return
      track('video_start', { video_status: 'start' });
      sentEvents.start = true;
    };
    const ended = () => {
      if (sentEvents.complete) return
      track('video_complete', { video_status: 'complete' });
      sentEvents.complete = true;
    };
    const timeupdate = () => {
      if (player.paused) return;
      const percent = Math.round((player.currentTime / player.duration) * 100);
      if (percent > maxPercent) {
        maxPercent = percent;
        if (percent >= 50 && !sentEvents.progress50) {
          sentEvents.progress50 = true;
          track('apx_video_progress', { video_percent: 50 });
        }
      }
    };
    player.addEventListener('play', play);
    player.addEventListener('ended', ended);
    player.addEventListener('timeupdate', timeupdate);

    return () => {
      player.removeEventListener('play', play);
      player.removeEventListener('ended', ended);
      player.removeEventListener('timeupdate', timeupdate);
    };
  }, deps);
};
```

また、本アプリケーションではGA4を採用しました。従来のUAではBigQueryとデータ連携するにはGA360を契約する必要がありましたが、GA4では有料プランでなくてもBigQuery連携が可能になっています。
BigQueryでデータを集計し、DataStudioで集計したデータを表示することで、かなり手軽に分析ができるようになりました。

## 運用フローの整備
アプリケーションを作り直すだけならともかく、画面設計も変更しているために運用フローを見直す必要がありました。このフローは長らく放置されていたので今回のリニューアルに合わせて、運用フローを作り変えています。

ドタバタしやすい校了前に行う情報の受け渡し、URL・QRコードの生成といったやり取りのルールを決めていくことに加えて、コロナ禍でフルリモートだったのもあり、Slackでのメッセージ、Zoomを使ったコミュニケーションだけで決めていくのは大変でした。

開発チーム向けにはNotionを使ったやり取りの管理テーブルの作成、アップロードを行うCLIツールの整備、公開フローを書いたREADMEの整備等を行いました。
公開直後は自分がネックになってしまうフローでしたが、開発チーム向けの情報を整備したことで自分の手を離れても運用が回るようになっています。

## おわりに
期間が限られている中しっかりリリースまで辿り着けたのは、フレームワーク選定やCloudrinaryの採用といった開発チームで培ってきたやり方であったり、Vercelを即日採用できる環境が大きいと思っています。

とくに今回で良かったのが、Vercel採用の判断だと思っています。
普段であれば、AWSの運用担当に本番環境、ステージング環境を用意してもらうことになります。しかし、Vercelは連携するだけで本番環境がたち、Pull Requestが立つたびに、検証環境が生成されます。もちろんVercelへのロックインは気をつける必要はありますが、自前でビルド設定を用意する必要もなくアプリケーションの開発に専念できるのはとてもよいことだと思います。今回のような小さなアプリケーションに対しては、非常にコスパがいいように感じました。

また、短い期間の開発でしたがいい影響も出てきており、ボトムアップのDXを体験しているようです。具体的には次のような変化がありました。

* ユーザーの間違えた行為が紙面の作り方に影響を与えるようになった
* 同じ仕組みを利用し、写真集にも動画特典がつくようになった
* 編集部とのやり取りがメールからSlackに変わった

こういった内部の変化を感じながらも、読者やユーザーに対してより良い体験を与えられるように「やっていき」の気持ちを持ち続けていきたいと思います。

=> [We are hiring](https://jp.indeed.com/cmp/Kodanshatech%E5%90%88%E5%90%8C%E4%BC%9A%E7%A4%BE)