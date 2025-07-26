# json-lib-2.4

コアJSON型: JSONObject、JSONArray、JSONNullがJSONの基本型を表現<br>
設定管理: JsonConfigでシリアライゼーション/デシリアライゼーションの動作を制御<br>
プロセッサーシステム: カスタム変換ロジックを実装するための各種プロセッサー<br>
フィルターシステム: プロパティの除外や変換を制御するフィルター<br>
イベントシステム: シリアライゼーション過程でのイベント通知<br>
柔軟な変換: JavaオブジェクトとJSON間の双方向変換をサポート<br>
このライブラリは、JavaオブジェクトとJSON間の変換を非常に柔軟かつ設定可能な形で提供しています。<br>

![主要クラス図](./主要クラス図.svg)

主要クラスの処理のシーケンス図
1. JavaオブジェクトからJSONObjectへの変換シーケンス
```mermaid
sequenceDiagram
    participant Client as クライアント
    participant JSONObject as JSONObject
    participant JsonConfig as JsonConfig
    participant AbstractJSON as AbstractJSON
    participant JsonValueProcessor as JsonValueProcessor
    participant JsonBeanProcessor as JsonBeanProcessor
    participant PropertyFilter as PropertyFilter

    Client->>JSONObject: fromObject(bean)
    JSONObject->>JSONObject: fromObject(bean, new JsonConfig())
    
    alt beanがnullの場合
        JSONObject->>JSONObject: JSONNull.getInstance()
    else beanがJSONStringの場合
        JSONObject->>JSONObject: _fromJSONString(string, config)
    else beanがStringの場合
        JSONObject->>JSONObject: _fromString(str, config)
    else beanがMapの場合
        JSONObject->>JSONObject: _fromMap(map, config)
    else beanがBeanの場合
        JSONObject->>JSONObject: _fromBean(bean, config)
    end

    Note over JSONObject: Bean処理の場合
    JSONObject->>JsonConfig: findJsonBeanProcessor(beanClass)
    alt BeanProcessorが登録されている場合
        JSONObject->>JsonBeanProcessor: processBean(bean, config)
        JsonBeanProcessor-->>JSONObject: JSONObject
    else デフォルト処理
        JSONObject->>JSONObject: defaultBeanProcessing(bean, config)
        JSONObject->>JsonConfig: getMergedExcludes(beanClass)
        JSONObject->>JsonConfig: getJsonPropertyFilter()
        
        loop 各プロパティに対して
            JSONObject->>PropertyFilter: apply(source, name, value)
            alt フィルターで除外される場合
                Note over JSONObject: プロパティをスキップ
            else 処理を続行
                JSONObject->>JsonConfig: findJsonValueProcessor(beanClass, type, key)
                alt ValueProcessorが登録されている場合
                    JSONObject->>JsonValueProcessor: processObjectValue(key, value, config)
                    JsonValueProcessor-->>JSONObject: 変換された値
                end
                JSONObject->>AbstractJSON: _processValue(value, config)
                JSONObject->>JSONObject: element(key, processedValue)
            end
        end
    end
    
    JSONObject-->>Client: JSONObject
```

2. JSONObjectからJavaオブジェクトへの変換シーケンス
```mermaid
sequenceDiagram
    participant Client as クライアント
    participant JSONObject as JSONObject
    participant JsonConfig as JsonConfig
    participant PropertyFilter as PropertyFilter
    participant PropertyNameProcessor as PropertyNameProcessor
    participant PropertySetStrategy as PropertySetStrategy

    Client->>JSONObject: toBean(jsonObject, config)
    
    alt jsonObjectがnullまたはnullObjectの場合
        JSONObject-->>Client: null
    else 処理を続行
        JSONObject->>JsonConfig: getRootClass()
        JSONObject->>JsonConfig: getClassMap()
        
        alt rootClassが指定されている場合
            JSONObject->>JsonConfig: getNewBeanInstanceStrategy()
            JSONObject->>NewBeanInstanceStrategy: newInstance(beanClass, jsonObject)
            NewBeanInstanceStrategy-->>JSONObject: beanインスタンス
        else
            JSONObject->>JSONObject: toBean(jsonObject) // DynaBean作成
        end
        
        JSONObject->>JsonConfig: getJavaPropertyFilter()
        
        loop JSONObjectの各プロパティに対して
            JSONObject->>JSONObject: get(name)
            JSONObject->>PropertyFilter: apply(bean, name, value)
            alt フィルターで除外される場合
                Note over JSONObject: プロパティをスキップ
            else 処理を続行
                JSONObject->>JsonConfig: findJavaPropertyNameProcessor(beanClass)
                alt PropertyNameProcessorが登録されている場合
                    JSONObject->>PropertyNameProcessor: processPropertyName(beanClass, key)
                    PropertyNameProcessor-->>JSONObject: 変換されたプロパティ名
                end
                
                JSONObject->>JsonConfig: getPropertySetStrategy()
                JSONObject->>PropertySetStrategy: setProperty(bean, key, value, config)
                PropertySetStrategy->>PropertySetStrategy: _setProperty(bean, key, value)
            end
        end
    end
    
    JSONObject-->>Client: Javaオブジェクト
```

3. JSONArrayの処理シーケンス
```mermaid
sequenceDiagram
    participant Client as クライアント
    participant JSONArray as JSONArray
    participant JsonConfig as JsonConfig
    participant AbstractJSON as AbstractJSON
    participant JsonValueProcessor as JsonValueProcessor

    Client->>JSONArray: fromObject(collection)
    JSONArray->>JSONArray: fromObject(collection, new JsonConfig())
    
    alt collectionが配列の場合
        JSONArray->>JSONArray: _fromArray(array, config)
    else collectionがCollectionの場合
        JSONArray->>JSONArray: _fromCollection(collection, config)
    else collectionがEnumの場合
        JSONArray->>JSONArray: _fromArray(enum, config)
    end
    
    Note over JSONArray: コレクション処理の場合
    JSONArray->>AbstractJSON: fireArrayStartEvent(config)
    
    loop コレクションの各要素に対して
        JSONArray->>JsonConfig: findJsonValueProcessor(elementType)
        alt ValueProcessorが登録されている場合
            JSONArray->>JsonValueProcessor: processArrayValue(element, config)
            JsonValueProcessor-->>JSONArray: 変換された値
        end
        JSONArray->>AbstractJSON: _processValue(element, config)
        JSONArray->>JSONArray: add(processedElement)
        JSONArray->>AbstractJSON: fireElementAddedEvent(index, element, config)
    end
    
    JSONArray->>AbstractJSON: fireArrayEndEvent(config)
    JSONArray-->>Client: JSONArray
```

4. JSONSerializerの統合シーケンス
```mermaid
sequenceDiagram
    participant Client as クライアント
    participant JSONSerializer as JSONSerializer
    participant JSONObject as JSONObject
    participant JSONArray as JSONArray
    participant JsonConfig as JsonConfig

    Note over Client, JsonConfig: Java → JSON 変換
    Client->>JSONSerializer: toJSON(object, config)
    
    alt objectがnullの場合
        JSONSerializer->>JSONSerializer: JSONNull.getInstance()
    else objectがJSONStringの場合
        JSONSerializer->>JSONSerializer: toJSON(string, config)
    else objectがStringの場合
        JSONSerializer->>JSONSerializer: toJSON(string, config)
    else objectが配列またはCollectionの場合
        JSONSerializer->>JSONArray: fromObject(object, config)
        JSONArray-->>JSONSerializer: JSONArray
    else objectがMapまたはBeanの場合
        JSONSerializer->>JSONObject: fromObject(object, config)
        JSONObject-->>JSONSerializer: JSONObject
    end
    
    JSONSerializer-->>Client: JSON

    Note over Client, JsonConfig: JSON → Java 変換
    Client->>JSONSerializer: toJava(json, config)
    
    alt jsonがJSONArrayの場合
        JSONSerializer->>JsonConfig: getArrayMode()
        alt MODE_OBJECT_ARRAYの場合
            JSONSerializer->>JSONArray: toArray(jsonArray, config)
        else MODE_LISTの場合
            JSONSerializer->>JSONArray: toCollection(jsonArray, config)
        end
    else jsonがJSONObjectの場合
        JSONSerializer->>JSONObject: toBean(jsonObject, config)
    end
    
    JSONSerializer-->>Client: Javaオブジェクト
```

5. イベント処理シーケンス

```mermaid
sequenceDiagram
    participant Client as クライアント
    participant JsonConfig as JsonConfig
    participant AbstractJSON as AbstractJSON
    participant JsonEventListener as JsonEventListener

    Client->>JsonConfig: enableEventTriggering()
    JsonConfig->>JsonConfig: triggerEvents = true
    
    Client->>JsonConfig: addJsonEventListener(listener)
    JsonConfig->>JsonConfig: eventListeners.add(listener)
    
    Note over Client, JsonEventListener: JSON処理中
    AbstractJSON->>JsonConfig: isEventTriggeringEnabled()
    JsonConfig-->>AbstractJSON: true
    
    AbstractJSON->>JsonConfig: getJsonEventListeners()
    JsonConfig-->>AbstractJSON: listeners
    
    loop 各リスナーに対して
        AbstractJSON->>JsonEventListener: onObjectStart()
        JsonEventListener-->>AbstractJSON: 
        AbstractJSON->>JsonEventListener: onPropertySet(key, value, accumulated)
        JsonEventListener-->>AbstractJSON: 
        AbstractJSON->>JsonEventListener: onObjectEnd()
        JsonEventListener-->>AbstractJSON: 
    end
```


これらのシーケンス図は、json-lib-2.4の主要な処理フローを示しています：
Java→JSON変換: オブジェクトの型に応じた適切な処理とプロセッサーの適用
JSON→Java変換: 設定に基づく適切なJavaオブジェクトの生成
配列処理: コレクションや配列のJSONArrayへの変換
統合処理: JSONSerializerによる統一的な変換インターフェース
イベント処理: 処理過程でのイベント通知システム