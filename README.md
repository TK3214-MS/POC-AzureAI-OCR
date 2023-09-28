k# アーキテクチャー
今回のアーキテクチャーは以下の通りです。

![00](https://github.com/TK3214-MS/POC-AzureAI-OCR/assets/89323076/a3c190bf-2783-427b-adf8-fb91b08632b8)

簡単な流れは以下の通りです。

1. Power Apps よりレシートをアップロードする。
1. 入力されたレシートデータが base64 形式でPower Automate フローに渡される。
1. 入力されたレシートデータを元に SharePoint Online ドキュメントライブラリに作成される。
1. ドキュメントライブラリ上データを元に Azure Custom Vision が OCR 処理をする。
1. Azure Cosmos DB に OCR 結果が渡され、データ座標に則って配列順序をソートし、Power Apps に応答する。

# 構成手順
以下を構成、実装する事で動作確認用環境の構成が可能です。

## 事前準備
### Azure Computer Vision リソースの作成
1. [手順ページ](https://learn.microsoft.com/en-us/training/paths/create-computer-vision-solutions-azure-cognitive-services/)に則り、Azure Computer Vision リソースを作成します。

1. [Azure ポータル](https://portal.azure.com)より作成した Computer Vision リソースの [KEY1]と[ENDPOINT]値をコピーします。

### SharePoint Online ドキュメントライブラリの準備
1. ファイルをアップロードするドキュメントライブラリを作成します。

### Azure Cosmos DB リソースの作成
1. [手順ページ](https://learn.microsoft.com/ja-jp/azure/cosmos-db/nosql/quickstart-portal#create-account)に則り、Cosmos DB アカウントを作成します。

1. 以下を作成します。

    | 種別 | 名前 |
    | データベース名| POC-AAI |
    | コンテナ名 | ReadResult |
    | パーティションキー | /runId |

    ![01](https://github.com/TK3214-MS/POC-AzureAI-OCR/assets/89323076/c3884d2a-4147-42e4-b397-3754aaeef2b2)
    ※Time to Live 値を 1800 秒 (30分) で構成し、自動的にデータがクリアされるようにしています。

## 構成
### Power Apps の構成
以下手順を構成頂くと以下アプリが完成します。

![02](https://github.com/TK3214-MS/POC-AzureAI-OCR/assets/89323076/3dbcf342-0b77-4c0d-a0bb-0406a1222982)

| No | コントロール種別 | コントロール名 |
| ---- | ---- | ---- |
| ① | 添付ファイル | DataCardValue3 |
| ② | ボタン | Button1 |
| ③ | テキストラベル | Label1 |

※添付ファイルコントロールを追加するには、一度編集フォームを追加、適当な SharePoint Online リストをデータソースに設定し添付ファイルコントロール以外のコントロールを削除します。

1. 添付ファイルコントロールへのプロパティ追加

    | プロパティ名 | 値 |
    | ---- | ---- |
    | Items | Blank() |
    | MaxAttachments | 1 |

1. ボタンコントロールへのプロパティ追加

    | プロパティ名 | 値 |
    | ---- | ---- |
    | OnSelect | Set(DesiredOutput,<br>'POC-AzureAI-OCR'.Run(<br>    {<br>        contentBytes: First(DataCardValue3.Attachments).Value,<br>        name: First(DataCardValue3.Attachments).Name<br>    }<br>)<br>); |

1. テキストラベルコントロール(AudioJSON)へのプロパティ追加

    | プロパティ名 | 値 |
    | ---- | ---- |
    | Text | DesiredOutput.text |

    > [!NOTE]<br>
    > 複数のエラーが表示されますが、Power Automate フローを関連付けると解消されます。

### Power Automate の構成
以下手順を構成頂くと以下フローが完成します。

![03](https://github.com/TK3214-MS/POC-AzureAI-OCR/assets/89323076/17cde655-e07e-47a2-bdad-218380e088d1)

1. Power Apps (V2) トリガーの構成

    ![04](https://github.com/TK3214-MS/POC-AzureAI-OCR/assets/89323076/fbf6f1d3-9e7b-4ec9-8a9c-360ef8b841d6)

    | 種別 | 名前 | 値 |
    | ---- | ---- | ---- |
    | ファイル | FileContent | ファイルまたはイメージを選択してください |

1. ファイルの作成アクションの構成

    ![05](https://github.com/TK3214-MS/POC-AzureAI-OCR/assets/89323076/05b7f753-5a5a-495b-997c-d58a65be364f)

    | サイトのアドレス | フォルダーのパス | ファイル名 | ファイル コンテンツ |
    | ---- | ---- | ---- | ---- |
    | 作成したライブラリが存在するサイトを選択 | ライブラリへのパス | triggerBody()['file']['name'] | triggerBody()['file']['contentBytes'] |

1. ファイルコンテンツの取得アクションの構成

    ![06](https://github.com/TK3214-MS/POC-AzureAI-OCR/assets/89323076/2d84adf1-95a3-43de-a78e-9961ad526318)

    | サイトのアドレス | ファイル識別し |
    | ---- | ---- |
    | 作成したライブラリが存在するサイトを選択 | outputs('ファイルの取得)?['body/Id'] |

1. HTTP (Computer Vision OCR 要求) アクションの構成

    ![07](https://github.com/TK3214-MS/POC-AzureAI-OCR/assets/89323076/175e1a12-782d-4eb0-be91-fc85ed140952)

    | 方法 | URI | ヘッダー | 本文 |
    | ---- | ---- | ---- | ---- |
    | POST | https://**ENDPOINT**.cognitiveservices.azure.com/vision/v3.2/read/analyze?language=ja&readingOrder=natural&model-version=latest | Content-Type：application/octet-stream<br>Ocp-Apim-Subscription-Key：**KEY** | {<br>  "$content-type": "image/jpeg",<br>  "$content": "@{body('ファイル_コンテンツの取得')?['body']}"<br>} |


1. JSON の解析 アクションの構成

    ![08](https://github.com/TK3214-MS/POC-AzureAI-OCR/assets/89323076/17ef2c41-681f-4c37-921d-e48a56d69f22)

    | コンテンツ | スキーマ |
    | ---- | ---- |
    | @{outputs('Computer_Vision_OCR_要求')['headers']} | 以下を参照 |
    
    ```json
    {
        "type": "object",
        "properties": {
            "Operation-Location": {
                "type": "string"
            },
            "x-envoy-upstream-service-time": {
                "type": "string"
            },
            "CSP-Billing-Usage": {
                "type": "string"
            },
            "apim-request-id": {
                "type": "string"
            },
            "Strict-Transport-Security": {
                "type": "string"
            },
            "X-Content-Type-Options": {
                "type": "string"
            },
            "x-ms-region": {
                "type": "string"
            },
            "Date": {
                "type": "string"
            },
            "Content-Length": {
                "type": "string"
            }
        }
    }
    ```

1. 待ち時間アクションの構成

    ![09](https://github.com/TK3214-MS/POC-AzureAI-OCR/assets/89323076/429cb2a9-aa52-40da-ad1f-036589894aed)

1. HTTP (Computer Vision 処理状況取得) アクションの構成

    ![10](https://github.com/TK3214-MS/POC-AzureAI-OCR/assets/89323076/b550a45c-c372-46c2-8fd3-947be09708cb)

    | 方法 | URI | ヘッダー |
    | ---- | ---- | ---- |
    | GET | @{body('JSON_の解析')?['Operation-Location']} | Ocp-Apim-Subscription-Key：**[Speech Service Key]** |

1. 選択アクションの構成

    ![11](https://github.com/TK3214-MS/POC-AzureAI-OCR/assets/89323076/61fd5db2-a634-47a2-9d0b-f921460831a7)

    | 開始 | マップ |
    | ---- | ---- |
    | @{body('Computer_Vision_処理状況取得')?['analyzeResult']?['readResults']?[0]?['lines']} | @item()?['text'] |

1. 結合アクションの構成

    ![12](https://github.com/TK3214-MS/POC-AzureAI-OCR/assets/89323076/dd21d252-fb2f-424c-870d-a9a7ad99e396)

    | 結合する配列 | 次を使用して結合 |
    | ---- | ---- |
    | @{body('選択')} | @{decodeUriComponent('%0A')} |

1. 作成アクションの構成

    ![13](https://github.com/TK3214-MS/POC-AzureAI-OCR/assets/89323076/fe5595d4-c692-4e0a-9d26-a63b09e018b4)

    | 入力 |
    | ---- |
    | @{guid()} |

1. Apply to each + ドキュメントを作成または更新 (V3) アクションの構成

    ![14](https://github.com/TK3214-MS/POC-AzureAI-OCR/assets/89323076/1e5d2575-7a59-442e-ace2-5a60fed6d4fa)

    | 以前の手順から出力を選択 |
    | ---- |
    | @{first(body('Computer_Vision_処理状況取得')?['analyzeResult']?['readResults'])?['lines']} |

    | body |
    | ---- |
    | {<br>  "id": "@{guid()}",<br>  "runId": "@{outputs('作成')}",<br>  "text": "@{items('Apply_to_each')?['text']}",<br>  "txtLeft": @{item()?['boundingBox']?[0]},<br>  "txtRight": @{item()?['boundingBox']?[2]},<br>  "txtTop": @{item()?['boundingBox']?[1]}<br>} |

1. ドキュメントをクエリする (V5) アクションの構成

    ![15](https://github.com/TK3214-MS/POC-AzureAI-OCR/assets/89323076/9125ef8e-e0d7-4032-86bc-fae7c6af48ab)

    | SQL 構文クエリ | パーティションキーの値 |
    | ---- | ---- |
    | select c.text, c.txtTop, c.txtLeft, c.txtRight from c order by c.txtTop asc, c.txtLeft | @{outputs('作成')} |

1. 変数の初期化アクションの構成

    ![16](https://github.com/TK3214-MS/POC-AzureAI-OCR/assets/89323076/69e958b3-cd8e-439d-8e89-6f1931437a5c)

1. Apply to each 2 コントロールの構成

    ![17](https://github.com/TK3214-MS/POC-AzureAI-OCR/assets/89323076/c13e2210-6746-4361-90c9-d2eb328ff278)

    | 以前の手順から出力を選択 |
    | ---- |
    | @{range(0,sub(length(outputs('ドキュメントをクエリする_(V5)')?['body']?['value']),1))} |

1. 文字列変数に追加アクションの構成

    ![18](https://github.com/TK3214-MS/POC-AzureAI-OCR/assets/89323076/d7f6017e-26a8-412f-84b7-1d875681fdfe)

    | 名前 | 値 |
    | ---- | ---- |
    | ScannedText | @{body('ドキュメントをクエリする_(V5)')?['value']?[item()]?['text']} |

1. 条件コントロールの構成

    ![19](https://github.com/TK3214-MS/POC-AzureAI-OCR/assets/89323076/ea8636da-8c92-400d-b65c-1e066e2303f8)

    | 条件 | 式 | 値 |
    | ---- | ---- | ---- |
    | body('ドキュメントをクエリする_(V5)')?['value']?[item()]?['txtRight'] | 次の値より大きい | @body('ドキュメントをクエリする_(V5)')?['value']?[item()]?['txtLeft'] |

### はいの場合

1. 文字列変数に追加 3 アクション の構成

    ![20](https://github.com/TK3214-MS/POC-AzureAI-OCR/assets/89323076/0ff8fd0a-410d-403f-8c9e-e9db4f1362d0)

    | 名前 | 値 |
    | ---- | ---- |
    | ScannedText | @{decodeUriComponent('%0D%0A')} |

#### いいえの場合
1. 文字列変数に追加 3 アクション の構成

    ![21](https://github.com/TK3214-MS/POC-AzureAI-OCR/assets/89323076/9008ccd2-983f-47bb-805c-4c95ef9c8807)

    | 名前 | 値 |
    | ---- | ---- |
    | ScannedText | 空白 |

1. 文字列変数に追加 4 アクションの構成

    ![22](https://github.com/TK3214-MS/POC-AzureAI-OCR/assets/89323076/8f0d68bd-4d24-4141-888d-a8cdab023ff1)

    | 名前 | 値 |
    | ---- | ---- |
    | ScannedText | @{last(body('ドキュメントをクエリする_(V5)')?['value'])?['text']} |

1. 作成 2 アクションの構成

    ![23](https://github.com/TK3214-MS/POC-AzureAI-OCR/assets/89323076/aedf918c-3e96-4615-a9ef-392ee420840b)

    | 入力 |
    | ---- |
    | @{variables('ScannedText')} |

1. PowerApp または Flow に応答する アクションの構成

    ![24](https://github.com/TK3214-MS/POC-AzureAI-OCR/assets/89323076/3c888a46-5c6e-4dc0-b864-454fec804591)

    | 形式 | 名前 | 値 |
    |---- | ---- | ---- |
    | 文字列 | text | @outputs('作成_2') |

## 動作確認
1. [こちら](https://learn.microsoft.com/ja-jp/power-apps/maker/canvas-apps/using-logic-flows#add-a-flow-to-an-app)を参考に Power Apps 上で作成した Power Automate フローを関連付けます。

1. Power Apps からレシートをアップロードします。

1. レシート内容がソートされたテキスト値で返されれば動作確認成功です。