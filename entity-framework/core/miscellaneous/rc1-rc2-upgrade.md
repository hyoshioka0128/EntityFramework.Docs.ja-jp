---
title: EF Core 1.0 RC1 から RC2-EF Core へのアップグレード
author: rowanmiller
ms.date: 10/27/2016
ms.assetid: 6d75b229-cc79-4d08-88cd-3a1c1b24d88f
uid: core/miscellaneous/rc1-rc2-upgrade
ms.openlocfilehash: 5300fe459ec2b8ab9bb573c7284b009249071d65
ms.sourcegitcommit: c9c3e00c2d445b784423469838adc071a946e7c9
ms.translationtype: MT
ms.contentlocale: ja-JP
ms.lasthandoff: 07/18/2019
ms.locfileid: "68306462"
---
# <a name="upgrading-from-ef-core-10-rc1-to-10-rc2"></a>EF Core 1.0 RC1 から 1.0 RC2 へのアップグレード

この記事では、RC1 パッケージでビルドされたアプリケーションを RC2 に移動するためのガイダンスを提供します。

## <a name="package-names-and-versions"></a>パッケージ名とバージョン

RC1 と RC2 の間では、"Entity Framework 7" から "Entity Framework Core" に変更されました。 この投稿の変更の理由の詳細については、 [Scott サバイバル selman を](http://www.hanselman.com/blog/ASPNET5IsDeadIntroducingASPNETCore10AndNETCore10.aspx)参照してください。 この変更により、パッケージ名はから`EntityFramework.*`に`Microsoft.EntityFrameworkCore.*`変わり、バージョンはから`7.0.0-rc1-final` ( `1.0.0-rc2-final`または`1.0.0-preview1-final`ツール) に変更されました。

**RC1 パッケージを完全に削除し、RC2 をインストールする必要があります。** いくつかの一般的なパッケージのマッピングを次に示します。

| RC1 パッケージ                                               | RC2 同等                                                       |
|:----------------------------------------------------------|:---------------------------------------------------------------------|
| EntityFramework. Microsoft Sqlserver 7.0.0-rc1-final | Microsoft EntityFrameworkCore. SqlServer 1.0.0-rc2-final      |
| EntityFramework-rc1-final | Microsoft EntityFrameworkCore. Sqlite 1.0.0-rc2-final      |
| EntityFramework7.Npgsql                   3.1.0-rc1-3     | NpgSql. EntityFrameworkCore. Postgres<to be advised>      |
| EntityFramework. SqlServerCompact35 7.0.0-rc1-final | EntityFrameworkCore. SqlServerCompact35 1.0.0-rc2-final      |
| EntityFramework. SqlServerCompact40 7.0.0-rc1-final | EntityFrameworkCore. SqlServerCompact40 1.0.0-rc2-final      |
| EntityFramework.InMemory                  7.0.0-rc1-final | Microsoft EntityFrameworkCore. InMemory-rc2-final      |
| EntityFramework. IBMDataServer-beta1     | RC2 ではまだ使用できません                                            |
| EntityFramework. Commands 7.0.0-final | Microsoft EntityFrameworkCore. ツール 1.0.0-preview1-final |
| EntityFramework. 7.0.0-rc1-final | Microsoft EntityFrameworkCore. SqlServer. Design-rc2-final      |

## <a name="namespaces"></a>名前空間

パッケージ名と共に、名前空間`Microsoft.Data.Entity.*`は`Microsoft.EntityFrameworkCore.*`からに変更されました。 この変更は、の`using Microsoft.Data.Entity` `using Microsoft.EntityFrameworkCore`検索と置換を使用して処理できます。

## <a name="table-naming-convention-changes"></a>テーブルの名前付け規則の変更

RC2 では、クラス名だけではなく、マップ先のテーブル`DbSet<TEntity>`名として指定されたエンティティのプロパティ名を使用しました。 この変更の詳細については[、関連する発表](https://github.com/aspnet/Announcements/issues/167)に関する問題を参照してください。

既存の rc1 アプリケーションでは、rc1 の名前付け方法を維持するため`OnModelCreating`に、メソッドの先頭に次のコードを追加することをお勧めします。

``` csharp
foreach (var entity in modelBuilder.Model.GetEntityTypes())
{
    entity.Relational().TableName = entity.DisplayName();
}
```

新しい名前付け方法を採用する場合は、残りのアップグレード手順を正常に完了してから、コードを削除し、移行を作成してテーブルの名前変更を適用することをお勧めします。

## <a name="adddbcontext--startupcs-changes-aspnet-core-projects-only"></a>AddDbContext/Startup.cs Changes (ASP.NET Core プロジェクトのみ)

RC1 では、アプリケーションサービスプロバイダー `Startup.ConfigureServices(...)`に Entity Framework サービスを追加する必要がありました。

``` csharp
services.AddEntityFramework()
  .AddSqlServer()
  .AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(Configuration["ConnectionStrings:DefaultConnection"]));
```

RC2 では`AddEntityFramework()`、 `AddSqlServer()`、などの呼び出しを削除できます。

``` csharp
services.AddDbContext<ApplicationDbContext>(options =>
  options.UseSqlServer(Configuration["ConnectionStrings:DefaultConnection"]));
```

また、コンテキストオプションを受け取り、基本コンストラクターに渡すコンストラクターを派生コンテキストに追加する必要もあります。 このことが必要になるのは、背後で仕込んする恐ろしいマジックの一部を削除したためです。

``` csharp
public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
    : base(options)
{
}
```

## <a name="passing-in-an-iserviceprovider"></a>IServiceProvider を渡す

をコンテキスト`IServiceProvider`に渡す RC1 コードがある場合、これは別のコンストラクターパラメーターで`DbContextOptions`はなく、に移動されました。 サービス`DbContextOptionsBuilder.UseInternalServiceProvider(...)`プロバイダーを設定するには、を使用します。

### <a name="testing"></a>テスト

これを行う最も一般的なシナリオは、テスト時に InMemory データベースのスコープを制御することでした。 RC2 を使用した例については、更新された[テスト](testing/index.md)に関する記事をご覧ください。

### <a name="resolving-internal-services-from-application-service-provider-aspnet-core-projects-only"></a>アプリケーションサービスプロバイダーからの内部サービスの解決 (ASP.NET Core プロジェクトのみ)

ASP.NET Core アプリケーションがあり、EF でアプリケーションサービスプロバイダーから内部サービスを解決する必要がある場合は、次のよう`AddDbContext`に構成できるのオーバーロードがあります。

``` csharp
services.AddEntityFrameworkSqlServer()
  .AddDbContext<ApplicationDbContext>((serviceProvider, options) =>
    options.UseSqlServer(Configuration["ConnectionStrings:DefaultConnection"])
           .UseInternalServiceProvider(serviceProvider));
```

> [!WARNING]  
> 内部の EF サービスをアプリケーションサービスプロバイダーに結合する理由がない限り、EF が独自のサービスを内部で管理できるようにすることをお勧めします。 これを行う主な理由は、EF が内部で使用するサービスをアプリケーションサービスプロバイダーを使用して置き換えることです。

## <a name="dnx-commands--net-cli-aspnet-core-projects-only"></a>DNX コマンド = > .NET CLI (ASP.NET Core プロジェクトのみ)

ASP.NET 5 プロジェクト用の`dnx ef`コマンドを以前に使用したことがある場合`dotnet ef`は、コマンドに移動しました。 同じコマンド構文が引き続き適用されます。 構文情報に`dotnet ef --help`はを使用できます。

コマンドの登録方法は RC2 で変更されています。 DNX は .NET CLI に置き換えられているためです。 コマンドがの`tools`セクションに登録される`project.json`ようになりました。

``` json
"tools": {
  "Microsoft.EntityFrameworkCore.Tools": {
    "version": "1.0.0-preview1-final",
    "imports": [
      "portable-net45+win8+dnxcore50",
      "portable-net45+win8"
    ]
  }
}
```

> [!TIP]  
> Visual Studio を使用する場合は、パッケージマネージャーコンソールを使用して ASP.NET Core プロジェクトに対して EF コマンドを実行できるようになりました (これは RC1 ではサポートされていません)。 このためには、 `tools` 「」の`project.json` 「」セクションにコマンドを登録する必要があります。

## <a name="package-manager-commands-require-powershell-5"></a>パッケージマネージャーのコマンドには PowerShell 5 が必要です

Visual Studio のパッケージマネージャーコンソールで Entity Framework コマンドを使用する場合は、PowerShell 5 がインストールされていることを確認する必要があります。 これは、次のリリースで削除される一時的な要件です (詳細については、「[問題 #5327](https://github.com/aspnet/EntityFramework/issues/5327) 」を参照してください)。

## <a name="using-imports-in-projectjson"></a>プロジェクトの json での "インポート" の使用

EF Core の依存関係の中には、まだ .NET Standard をサポートしていないものがあります。 .NET Standard および .NET Core プロジェクトの EF Core には、一時的な回避策として、プロジェクトに "imports" を追加することが必要になる場合があります。

EF を追加すると、次のエラーメッセージが表示されます。

``` Console
Package Ix-Async 1.2.5 is not compatible with netcoreapp1.0 (.NETCoreApp,Version=v1.0). Package Ix-Async 1.2.5 supports:
  - net40 (.NETFramework,Version=v4.0)
  - net45 (.NETFramework,Version=v4.5)
  - portable-net45+win8+wp8 (.NETPortable,Version=v0.0,Profile=Profile78)
Package Remotion.Linq 2.0.2 is not compatible with netcoreapp1.0 (.NETCoreApp,Version=v1.0). Package Remotion.Linq 2.0.2 supports:
  - net35 (.NETFramework,Version=v3.5)
  - net40 (.NETFramework,Version=v4.0)
  - net45 (.NETFramework,Version=v4.5)
  - portable-net45+win8+wp8+wpa81 (.NETPortable,Version=v0.0,Profile=Profile259)
```

この回避策は、ポータブルプロファイル "net451 + win8" を手動でインポートすることです。 これにより、NuGet は、これに一致するこのバイナリを .NET Standard と互換性のあるフレームワークとして処理します。 "Net451 + win8" は、.NET Standard と互換性が 100% ではありませんが、PCL から .NET Standard への移行に十分対応しています。 EF の依存関係が最終的に .NET Standard にアップグレードされると、インポートを削除できます。

配列構文では、複数のフレームワークを "インポート" に追加できます。 その他のインポートは、プロジェクトにライブラリを追加する場合に必要になることがあります。

``` json
{
  "frameworks": {
    "netcoreapp1.0": {
      "imports": ["dnxcore50", "portable-net451+win8"]
    }
  }
}
```

「[問題 #5176](https://github.com/aspnet/EntityFramework/issues/5176)」を参照してください。
