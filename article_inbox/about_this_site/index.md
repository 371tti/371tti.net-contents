---
title: About This Site
description: このサイトについて
authors:
- 371tti
is_complete: true
---

このサイトは371ttiが運営する個人サイトであり、目的は情報発信と自己表現です。  
主に技術的な内容を中心に、様々なトピックについて記事を投稿しています。

# システムと技術スタック

本サイトのシステムは371ttiが[Rust](https://www.rust-lang.org/)で開発したものです。github上で公開しており、システムとコンテンツの2つのリポジトリに分かれています。  
- [system repo](https://github.com/371tti/371tti.net)
- [content repo](https://github.com/371tti/371tti.net-contents)
 
以下はこのサイトの主要な技術スタックと関連プロジェクトです。  
- web backend server: [371tti@kurosabi](https://github.com/371tti/kurosabi)
- index and search: [371tti@tf-idf-vectorizer](https://github.com/371tti/tf-idf-vectorizer)
- md to html render: [markdown-rs](https://docs.rs/markdown/latest/markdown/)

Rustで開発した理由は以下の通り
- 低リソース環境で安定した運用が条件だったため。
- ドキュメントサーバー程度の規模であれば大して複雑ではなく、HTML, CSS, JS, Rustを効率良く活用すればすばらしいものが作れると考えたため。
- モダンWeb技術はAIの発展で数年サイクルで大きく変化しており、学生として学習期間にある自分にとっては、安定している基礎技術を十分に活用する能力のほうが重要だから。
- (私のアンチframework, ラブRust精神に基づいて。)

# ロードマップ
今後の予定としては、以下のような機能追加や改善を考えてるけどどうなるかはわからん
- [x] v2.x.xへの移行: 静的ページ配信からドキュメントサーバーへの完全移行を完了させる。
- [x] UI/UXの改善: ユーザーインターフェースの見直しとユーザビリティの向上を図る。
- [x] 管理機能の強化: システムとコンテンツの分離、セッション管理の実装、アクセス制御の強化、自動更新機能の追加など。
- [ ] パフォーマンスの最適化: レンダリングのキャッシュ実装や、サーバーのレスポンス速度の向上を目指す。
- [ ] 強力な検索機能の実装: 検索機能の実装と最適化。 
- [ ] Analyzerの強化: ユーザビリティ向上のためのAnalyzerの改善と機能追加。
- [ ] アカウント機能の実装: ユーザー登録、ログイン、プロフィール管理などのアカウント関連機能の実装。 
