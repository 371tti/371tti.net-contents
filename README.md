# 371tti.net Contents Repository

このディレクトリは **371tti.net のコンテンツ用リポジトリ** です。  
記事・ページ本文・静的アセットなど、サイト表示に必要なコンテンツを管理します。

## Pull Request について

改善提案・修正提案の **Pull Request を歓迎しています（お待ちしています）**。  
誤字修正、小さな改善、記事追加など気軽に送ってください。

## 自動更新
main ブランチへのマージ後、システムにより自動でサイトへ反映されます。  
371tti.net のフッターにある content hash を見ると、最新のコンテンツが反映されているか確認できます。

## 対応メタ情報
htnl又はmdにおいて先頭にyamlでメタ情報を置けます  
例:
```yaml
---
title: サンプル記事1
authors: 
- 371tti
tags:
- sample
is_complete: true
description: サンプル記事についてのちょっとした説明
---
```