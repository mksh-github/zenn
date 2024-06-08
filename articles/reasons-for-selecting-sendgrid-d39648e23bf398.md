---
title: "【備忘録】SendGrid選定理由"
emoji: "✉️"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["技術選定", "sendgrid", "python", "python3"]
published: true
---

# 概要

とあるプロジェクトのメール通知機能にて、SendGridを採用した理由に関する社内向けの備忘録です。具体的なプロジェクトの詳細は抽象化しています。

# メール配信の見込み数

リリース初期段階では、月間3,730通のメール配信を見込んでいます。

## 内訳

- ユーザ数: 120名（リリース初期）
- 定時通知メール: 1回/日
- 不定期通知メール: 平均10回/月

各ユーザに対して、定期および不定期の通知メールを送信します。

# クライアントからの要求

- メールを通知できるようにして欲しい

# 設計・実装方針

- 開発工数を極力減らす
- コードの保守を極力減らす
- Python標準ライブラリによる自前実装、またはメール配信サービスを利用する

# 自前実装 vs サービス利用

## 結論

メール配信サービスを利用する

## 理由

最大の理由は、ハードバウンスやソフトバウンス対策の観点から、Pythonでの自前実装が適切でないためです。

初期段階ではこれらの対策を実装しなくても問題はありませんが、仮に実装したとしても、再試行処理が適切でない場合、届かないメールへの送信試行が大量に行われることになります。結果として送信元IPの評価（IPレピュテーションスコア）が低下し、大きなリスクを伴う可能性があります。

そのためメール配信サービスを利用することで、このリスクを回避します。

# メール配信サービスの選定

## 選定基準

<!-- 以下の基準は開発側で仮で設定したものですが、必要に応じてクライアントと合意を得ることで正式な要件となります。（実際に正式な要件に落とし込む場合は整理が必要です） -->

- トランザクションメールの配信が可能であること
- バウンス対策が施されていること
- バウンスの管理が容易であり、事前に用意されたコンソール画面で管理できること
- APIに対応していること
- 初期段階では無料枠で運用可能であり、プランを上げた場合も考慮すること
- 情報量が豊富であること

## 結論

SendGrid

## 競合サービス比較

主な競合サービスとしては以下が挙げられます。

- SendGrid
- Mailgun
- AWS SES
- Mandrill (Mailchimp)

選定基準を踏まえた結果、最終候補としてSendGridとMailgunが残りました。どちらもほぼ同様の機能を提供しており、機能面ではどちらを採用しても選定基準を満たせます。

### SendGridの優位点

- コスト面: 送信数に応じたプランの変更において、**特に共有IPを利用する場合**、SendGridはコスト面で優れている
- 情報量と実績: SendGridはMailgunと比較して情報量が豊富で、実績も多い
- コスト予測: SendGridは円建てで料金が設定されており、為替レートの影響を受けない

なおSendGridのフリープランは送信数の上限を越えると送信できなくなるため注意が必要です（詳細は公式HP https://sendgrid.kke.co.jp/plan/ から最新情報を確認してください）。

一方、Mailgunはフリープランであっても送信数上限を超えた場合は課金となり、実際には送信数に上限がないという点が異なります。

### 選定から外れたサービス

- AWS SES
  - 社内でのAWSの導入が進んでいないため候補から外れました。またバウンス対策の設定が複雑になる可能性も理由の一つです。
- Mandrill (Mailchimp)
  - Mailchimpアカウントの保有が必要であり、無料枠がないため候補から外れました。

## 送信元IPアドレスの固定

現時点ではメール通知の可用性に関連する要件はありませんが、本格的に運用することとなった場合は可用性に関連して固定IPの検討も必要です[^1]。

この場合、SendGridおよびMailgun共にプランを上げる必要があります。
以下、固定IPが可能なプランを以下に示します（2022年11月8日時点）。

| サービス名 | プラン | 料金 |
| --- | --- | --- |
| SendGrid | Pro 100K | $89.95〜 |
| Mailgun | Foundation 100K | $75〜 |

なお共有IPのまま運用を続ける場合、自前実装よりも確率は低いと思います（根拠はありません）が、IPレピュテーションの低下やスロットリングなどのリスクがあります。

またコスト面を重視する場合、AWSの導入が進めばSESも視野に入ってきますが、SESでの固定IPに関連した調査と再比較が必要になります。

[^1]:[仕事でSendGridの安いEssentialsプランを使うのは辞めよう](https://qiita.com/rana_kualu/items/e3860b07a3919ae973a1)

# 実装例

## 要件

- Gmailで見た時にURLがクリックできること
- CCやBCCは用いず、一括送信を行った際にユーザに個別に送信されること

## コード例

上記要件を満たすコードの例を以下に示す。
なお、**以下コードはあくまでも確認用のコードのため取り扱いには注意**。

```python
import os
from typing import List
from sendgrid import SendGridAPIClient
from sendgrid.helpers.mail import Mail


SENDGRID_API_KEY = os.environ.get('_SENDGRID_API_KEY')


def send_email_via_sendgrid(
    sender_email: str,
    to_email: List[str] or str,
    subject: str,
    lines: List[str],
    is_multiple=True
) -> bool:
    message = Mail(
        subject=subject,
        plain_text_content='\n'.join(lines))

    message.from_email = sender_email
    message.add_to(to_email, is_multiple=is_multiple)

    sg_client = SendGridAPIClient(SENDGRID_API_KEY)
    response = sg_client.send(message)

    if response.status_code >= 400:
        return False

    return True


if __name__ == '__main__':
    SENDER_EMAIL = '送信元メールアドレス'

    result = send_email_via_sendgrid(
        SENDER_EMAIL,
        ['送信先メールアドレス1', '送信先メールアドレス2'],
        'SendGrid送信テスト（件名）',
        ['SendGrid送信テスト（本文）', '', 'リンク確認', 'https://www.google.co.jp/'],
        is_multiple=True)

    print(result)
```

以下、補足。

- `add_to` メソッドの引数 `is_multiple` を `True` にすることで個別に送信
- 送信元のアドレスは間違っていたとしても送信が可能なため注意
- SendGridのクリックトラッキングが不要な場合は無効化する（今回のようにプレーンテキストでURLを載せる場合、無効化しないとトラッキングURLがメール本文に入り込んでしまうのでとても怪しげなメールになります）
- 実装自体は1時間程度で可能（ドキュメントを見る時間含む）

# 参考

- 料金比較
  - [SendGrid pricing](https://sendgrid.kke.co.jp/plan/)
  - [mailgun pricing](https://www.mailgun.com/plans-and-pricing/)
- メール配信サービス比較
  - [Compare SendGrid vs Mailjet vs Mailgun](https://www.saasworthy.com/compare/mailgun-vs-mailjet-vs-sendgrid?pIds=173,1014,1554)
  - [Sendgrid vs Mandrill vs Mailgun – How to Choose?](https://mailtrap.io/blog/sendgrid-vs-mandrill-vs-mailgun/)
  - [Amazon SES vs SendGrid](https://mailtrap.io/blog/amazon-ses-vs-sendgrid/)
- IPレピュテーション関連
  - [送信ドメイン認証（SPF / DKIM / DMARC）の仕組みと、なりすましメール対策への活用法を徹底解説](https://ent.iij.ad.jp/articles/172/)
  - [固定IPアドレスを利用するメリットは何でしょうか？](https://support.sendgrid.kke.co.jp/hc/ja/articles/202688589)
  - [IPレピュテーションの基礎知識、メール到達率との関係を解説](https://baremail.jp/blog/2019/08/27/280/)
  - [メール送信のIPレピュテーションを向上させるには？](https://note.com/noriyuki_fujita/n/n19b2554f0e22)
- コーディング関連
  - [安心・便利に一斉送信するには？メール配信サービスをオススメする理由](https://sendgrid.kke.co.jp/blog/?p=14960)
  - [sendgrid/sendgrid-python GitHub](https://github.com/sendgrid/sendgrid-python)
  - [Python SendGrid V3 API wrapper example](https://sendgrid.kke.co.jp/docs/Integrate/Code_Examples/v3_Mail/python.html)
  - [SendGrid V3 API Documentation](https://docs.sendgrid.com/api-reference/how-to-use-the-sendgrid-v3-api/authentication)

# キーワード

- トランザクションメール
- IPレピュテーション
- バウンス
- 共有IP・固定IP（専用IP）
- API
- コスト
- 送信ドメイン認証
