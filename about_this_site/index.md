---
title: About This Site
description: このサイトについて
authors:
- 371tti
tags:
- site
- meta
- about
- index
is_complete: true
---

このサイトは371ttiが運営する個人サイトであり、目的は情報発信と自己表現です。  
ノリと勢いで個人ブログには過剰な機能をもりもりもりもりしてます  
主に技術的な内容を中心に、様々なトピックについて記事を投稿しています。  
コンテンツ部分はgithub上で管理されており、誰でも編集に参加できるのでぜひコントリビュートしてみてね?  

# システムと技術スタック

本サイトのシステムは371ttiが[Rust](https://www.rust-lang.org/)で開発したものです。github上で公開しており、システムとコンテンツの2つのリポジトリに分かれています。  
- [system repo](https://github.com/371tti/371tti.net)
- [content repo](https://github.com/371tti/371tti.net-contents)
 
以下はこのサイトの主要な技術スタックと関連プロジェクトです。  
- web backend server: [371tti@kurosabi](https://github.com/371tti/kurosabi)
- index and search: [371tti@tf-idf-vectorizer](https://github.com/371tti/tf-idf-vectorizer)
- session management: [371tti@sv-session](https://github.com/371tti/srv-session)
- md to html render: [markdown-rs](https://docs.rs/markdown/latest/markdown/)
- git integration: [gix](https://docs.rs/gix/)
- morphological analysis: [sudachi-rs](https://github.com/WorksApplications/sudachi.rs)

Rustで開発した理由は以下の通り
- 低リソース環境で安定した運用が条件だったため, 鯖が非力でも動くようにしたかった
- 魔改造, 独自機能をきれいに実装したかったから, 回りくどい実装は嫌ですよね？

# ロードマップ
今後の予定としては、以下のような機能追加や改善を考えてるけどどうなるかはわからん
- [x] v2.x.xへの移行: 静的ページ配信からドキュメントサーバーへの完全移行を完了させる。- [x] UI/UXの改善: ユーザーインターフェースの見直しとユーザビリティの向上を図る。
- [x] 管理機能の強化: システムとコンテンツの分離、セッション管理の実装、アクセス制御の強化、自動更新機能の追加など。
- [x] パフォーマンスの最適化: レンダリング結果のキャッシュ機構の実装や、サーバーのレスポンス速度の向上を目指す。
- [x] 強力な検索機能の実装: tf-idfベクトル化を活用した検索機能の実装と最適化。 
- [x] Analyzerの強化: ユーザビリティ向上のためのAnalyzerの改善と機能追加。
- [ ] アカウント機能の実装: ユーザー登録、ログイン、プロフィール管理などのアカウント関連機能の実装。 

