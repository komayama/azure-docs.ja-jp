---
title: インクルード ファイル
description: インクルード ファイル
services: storage
author: roygara
ms.service: storage
ms.topic: include
ms.date: 05/05/2019
ms.author: rogarana
ms.custom: include file
ms.openlocfilehash: 95c553d26a3e79b53106b933c629c5884c3e004c
ms.sourcegitcommit: 877491bd46921c11dd478bd25fc718ceee2dcc08
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 07/02/2020
ms.locfileid: "84466961"
---
| リソース | 移行先 | ハード制限 |
|----------|--------------|------------|
| リージョンあたりのストレージ同期サービス数 | 100 のストレージ同期サービス | はい |
| ストレージ同期サービスごとの同期グループ数 | 200 の同期グループ | はい |
| ストレージ同期サービスごとの登録サーバー数 | 99 サーバー | はい |
| 同期グループあたりのクラウド エンドポイント数 | 1 クラウド エンドポイント | はい |
| 同期グループあたりのサーバー エンドポイント数 | 50 サーバー エンドポイント | いいえ |
| サーバーあたりのサーバー エンドポイント数 | 30 個のサーバー エンドポイント | はい |
| 同期グループあたりのファイル システム オブジェクト数 (ディレクトリとファイル) | 1 億オブジェクト | いいえ |
| ディレクトリ内のファイル システム オブジェクト (ディレクトリとファイル) の最大数 | 500 万オブジェクト | はい |
| オブジェクト (ディレクトリとファイル) のセキュリティ記述子の最大サイズ | 64 KiB | はい |
| ファイル サイズ | 100 GiB | いいえ |
| 階層化するファイルの最小ファイル サイズ | V9:ファイル システムのクラスター サイズ (ダブルファイル システムクラスターのサイズ) に基づきます。 たとえば、ファイル システム クラスターのサイズが 4 KB の場合、ファイルの最小サイズは 8 KB になります。<br> V8 およびそれ以前:64 KiB  | はい |

> [!Note]  
> Azure File Sync エンドポイントは、Azure ファイル共有のサイズにスケールアップできます。 Azure ファイル共有のサイズの上限に達した場合、同期は操作できません。
