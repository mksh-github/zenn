---
title: "【備忘録】SendGrid選定理由"
emoji: "✉️"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["技術選定", "sendgrid", "python", "python3"]
published: true
---

# 概要

とあるプロジェクトのメール通知機能にてSendGridを採用した理由に関する社内向けの備忘録。実際のプロジェクトに関する情報は抽象化している。

# プロジェクトの規模感

SendGridを採用したプロジェクトでの見積もりは大まかに以下の通りとなる。

- ユーザ数120名（リリース初期）
- 1回/日の定時連絡メール
- 10回/月程度（平均）の不定期通知メール（リリース初期）

なお定時連絡および不定期の連絡はユーザそれぞれに対してメールが送信される。
またユーザ数は120名単位で増加する見込みあり。

上記より、リリース初期段階では3,730通/月のメール配信が見込まれる。

# メール通知機能に対する要求・要件

- クライアントからの要求は特になし
- 開発プロセスに対する社内的な要件
  - 開発工数を極力減らす
  - コードの保守を極力減らす
  - Python標準ライブラリによる自前実装、またはメール配信サービスを利用する

# 自前実装とメール配信サービスどちらにするか

結論から述べると、メール配信サービスを利用する。

理由は、当然ではあるが開発工数やコードの保守を極力減らすという観点からPythonでの自前実装は選択肢から外れ、また自前実装の場合、特に大量の配信を行ったり、エラー時の処理方法によっては送信元IPの評価（IPレピュテーションスコア）を失う可能性が高く、それらのリスクを軽減するためメール配信サービスを利用することとした。

# メール配信サービスの選定

## 選定基準

- トランザクションメールの配信
- バウンス対策が可能
- バウンスの管理のしやすさ（事前に用意されたコンソール画面で管理できる）
- API対応している
- 初期段階では無料枠で運用可能（プランを上げた場合も考慮する）
- 情報量の多さ

## 主な競合サービス

- SendGrid
- Mailgun
- AWS SES
- Mandrill (Mailchimp)

## 選定

選定要件を踏まえ、SendGridとMailgunが候補に挙がったが、いずれのメール配信サービスもほぼ同様の機能を保有しており、正直なところどちらを採用しても要件を満たすことができると考える。

しかし送信数に応じてプランを上げた場合、コスト面でSendGridが優れており（**特に共有IPを利用する場合**）、また比較して情報量や実績の多いであろうSendGridを利用する流れとなった。またSendGridは円建てでレートに左右されないため、料金が分かりやすいことなども挙げられた。

なおリリース後、送信上限である12,000通を越える時期はまだ見えないが、SendGridのフリープランは送信数の上限を越えると送信できなくなるため注意が必要。その点、Mailgunはフリープランであっても送信数上限を超えた場合は課金となり、実際には送信数に上限はないため安心かもしれない。

以下、選定から外れたサービスについて記述する。

- AWS SESは社内的な問題としてAWSの導入が進んでいないため候補から外れた。
- MandrillはMailchimpアカウントを保有している必要があること、それに関連して無料枠がないということから候補から外れた。

## 送信元IPアドレスの固定について

現時点では送信元IPアドレスの固定に関連するような要求・要件は特にないが、本格的に運用することとなった場合は固定IPを検討する必要が考えられ[^1]、その場合Mailgunの方がコスト面では若干優れる。
なおSendGridおよびMailgunでは、それぞれ以下に提示するプラン以上のプランが必要になる（2022年11月8日時点）。

| サービス名 | プラン | 料金 |
| --- | --- | --- |
| SendGrid | Pro 100K | $89.95〜 |
| Mailgun | Foundation 100K | $75〜 |

また詳しい調査は行っていないため憶測ではあるが、AWSの導入が進めばSESも視野に入ってくるため、コスト面を重視する場合は再度固定IPに関連した選定を行う必要が出てくる可能性はある。
その場合、より詳細なユースケースと、共有IPのまま運用を続けた場合のリスクについて考える必要がある。

[^1]:[仕事でSendGridの安いEssentialsプランを使うのは辞めよう](https://qiita.com/rana_kualu/items/e3860b07a3919ae973a1)

# 実装例

## 要件

- Gmailで見た時にURLがクリックできること
- CCやBCCは用いず、一括送信を行った際に個別に送信されること

## コード例

上記要件を満たすコードの例を以下に示す。
なお、以下コードはあくまでも確認用のコードのため取り扱いには注意。

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

    try:
        sg_client = SendGridAPIClient(SENDGRID_API_KEY)
        response = sg_client.send(message)
    except Exception:
        return False

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
- SendGridのクリックトラッキングが不要な場合は無効化する
  今回のようにプレーンテキストでURLを載せる場合、SendGridのクリックトラッキングを無効化しないと、トラッキングURLがメール本文に入り込んでしまうため、とても怪しげなメールとなってしまう。
- ドキュメントを見る時間含め、特に今回はプレーンテキストのメールだったため、実装自体は1時間程度で可能

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
