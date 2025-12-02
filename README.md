# Kotlin Backend Guidelines

Spring Boot WebFlux + Kotlin で OAuth 2.0 Resource Server を構築するための開発指針集

## 概要

このリポジトリは、Kotlin と Spring Boot WebFlux を使用した **OAuth 2.0 Resource Server（API サーバー）** を開発する際のベストプラクティスをまとめたガイドラインです。

実務で直面する課題に対する具体的な解決策と、一貫性のあるコードベースを維持するための指針を提供します。

## 前提条件

### 技術スタック

| 技術              | バージョン | 用途                            |
| ----------------- | ---------- | ------------------------------- |
| Kotlin            | 1.9+       | プログラミング言語              |
| Spring Boot       | 3.2+       | アプリケーションフレームワーク  |
| Spring WebFlux    | 6.1+       | リアクティブ Web フレームワーク |
| Kotlin Coroutines | 1.7+       | 非同期処理                      |
| R2DBC             | 1.0+       | リアクティブデータベース接続    |
| PostgreSQL        | 15+        | データベース                    |
| Spring Security   | 6.2+       | 認証・認可                      |

### アーキテクチャの前提

このガイドラインは以下のアーキテクチャを前提としています：

1. **OAuth 2.0 Resource Server**

   - 認証・認可は外部の Authorization Server（Keycloak、Auth0、Cognito 等）に委譲
   - Scope ベースのアクセス制御

2. **リアクティブスタック**

   - Spring WebFlux によるノンブロッキング処理
   - Kotlin Coroutines による非同期プログラミング
   - R2DBC によるリアクティブデータベースアクセス

3. **実践的なレイヤードアーキテクチャ**
   - Presentation（Controller）
   - Application（Service）
   - Domain（Entity、Repository Interface）
   - Infrastructure（Repository Implementation）

### 設計思想：段階的な発展を可能にする

このガイドラインは**初期構築段階での過度な抽象化を避け、実践的でシンプルな実装から始める**ことを重視しています。

#### 初期段階の方針

- **薄いドメイン層**: Entity は基本的にデータベーステーブルのマッピング（`@Table`付き）
- **シンプルな Service**: ビジネスロジックは Application 層の Service に集約
- **プリミティブ型中心**: Value Object は必要性が明確な場合のみ採用
- **Repository は Interface + Implementation**: 将来の実装切り替えに備えつつ、過度な抽象化は避ける

#### 発展可能な設計

プロジェクトの成長に応じて、以下のような発展が可能です：

- **ドメイン層の充実**: Rich Domain Model への移行（Entity にビジネスロジックを移動）
- **Value Object の導入**: 型安全性が必要な箇所で段階的に導入
- **Port/Adapter パターン**: 外部システム連携の複雑化に応じて導入
- **CQRS の部分適用**: 読み取り/書き込みの要件が乖離した場合に検討

**重要**: 最初から完璧な設計を目指すのではなく、**現在の要件に対して必要十分な設計**から始め、**リファクタリングしやすい構造**を維持することを目指します。

## ガイドライン目次

1. [プロジェクト構成](01-project-structure.md) - パッケージ構成、レイヤー設計、命名規則
2. [Kotlin イディオム](02-kotlin-idioms.md) - Kotlin らしい書き方、data class、sealed class
3. [依存性注入](03-dependency-injection.md) - DI の原則、テスタビリティ
4. [エラーハンドリング](04-error-handling.md) - 例外設計、RFC 7807 Problem Details
5. [API 設計](05-api-design.md) - RESTful API、リクエスト/レスポンス設計
6. [セキュリティ](06-security.md) - OAuth 2.0、RFC 6750、Scope ベース認可
7. [非同期・並行処理](07-concurrency.md) - Coroutines、Flow、WebClient
8. [テスト](08-testing.md) - 単体テスト、統合テスト、MockK
9. [設定・環境管理](09-configuration.md) - プロファイル、環境変数、機密情報管理

詳細は [目次](00_ToC.md) を参照してください。

## 特徴

### 実践的なガイドライン

- 抽象的な理論ではなく、実装可能な具体例を提示
- よくあるアンチパターンとその解決策を明記
- コピー&ペーストで使えるコードスニペット
- **初期段階ではシンプルに、必要に応じて発展可能な設計**

### 段階的な成長を支援

- 過度な抽象化を避けた実践的な初期実装
- プロジェクトの成熟度に応じたリファクタリング指針
- 薄いドメイン層から Rich Domain Model への移行パス
- 現在の要件に対して必要十分な設計から始める

### RFC/標準準拠

- **RFC 7807** (Problem Details for HTTP APIs) に準拠したエラーレスポンス
- **RFC 6750** (Bearer Token) に準拠した認証エラーハンドリング
- **OAuth 2.0** Resource Server としての正しい実装

### 一貫性のある設計

- プロジェクト全体で統一された命名規則
- レイヤー間の責務分離が明確
- テスタビリティを重視した設計
- リファクタリングしやすい構造

## 使い方

### 新規プロジェクトの場合

1. [01-project-structure.md](01-project-structure.md) からプロジェクト構成を確認
2. 各ガイドラインに従って実装を進める
3. チェックリストで実装漏れを確認

### 既存プロジェクトの場合

1. 現在の実装と各ガイドラインを比較
2. 優先度の高い項目から段階的に適用
3. チーム内で合意を取りながら進める

## 対象読者

- Kotlin でバックエンド API を開発するエンジニア
- Spring Boot から WebFlux への移行を検討している方
- OAuth 2.0 Resource Server の実装方法を学びたい方
- リアクティブプログラミングのベストプラクティスを知りたい方

## 非対象

このガイドラインでは以下は扱いません：

- Authorization Server の実装（Keycloak、Auth0 等の既存サービスを使用する前提）
- フロントエンド（SPA 等）の実装
- Spring MVC（同期処理）を使用した実装
- JPA/Hibernate を使用した実装

## ライセンス

このガイドラインは自由に参照・引用していただけます。
