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

6. JSONからXMLへの変換シーケンス
```mermaid
sequenceDiagram
    participant Client as クライアント
    participant XMLSerializer as XMLSerializer
    participant JSONObject as JSONObject
    participant JSONArray as JSONArray
    participant JSONNull as JSONNull
    participant Document as Document
    participant Element as Element
    participant XomSerializer as XomSerializer

    Client->>XMLSerializer: write(json)
    XMLSerializer->>XMLSerializer: write(json, null) // UTF-8エンコーディング
    
    alt jsonがJSONNullの場合
        XMLSerializer->>XMLSerializer: newElement(getRootName() || getObjectName())
        XMLSerializer->>Element: addAttribute("json_null", "true")
        XMLSerializer->>Document: new Document(root)
        XMLSerializer->>XMLSerializer: writeDocument(doc, encoding)
    else jsonがJSONArrayの場合
        XMLSerializer->>XMLSerializer: processJSONArray(jsonArray, root, expandableProperties)
        XMLSerializer->>Document: new Document(root)
        XMLSerializer->>XMLSerializer: writeDocument(doc, encoding)
    else jsonがJSONObjectの場合
        alt jsonObjectがnullObjectの場合
            XMLSerializer->>XMLSerializer: newElement(getObjectName())
            XMLSerializer->>Element: addAttribute("json_null", "true")
        else 通常のオブジェクト処理
            XMLSerializer->>XMLSerializer: processJSONObject(jsonObject, root, expandableProperties, true)
        end
        XMLSerializer->>Document: new Document(root)
        XMLSerializer->>XMLSerializer: writeDocument(doc, encoding)
    end
    
    Note over XMLSerializer: processJSONObjectの処理
    XMLSerializer->>XMLSerializer: newElement(getRootName() || getObjectName())
    
    loop JSONObjectの各プロパティに対して
        XMLSerializer->>XMLSerializer: processJSONValue(value, root, element, expandableProperties)
        
        alt valueがJSONArrayでexpandElementsまたはexpandablePropertiesに含まれる場合
            XMLSerializer->>XMLSerializer: 配列要素を展開して各要素を個別のXML要素として出力
        else 通常の値処理
            alt valueがJSONObjectの場合
                XMLSerializer->>Element: addAttribute("json_class", "object")
                XMLSerializer->>XMLSerializer: processJSONObject(value, element, expandableProperties, false)
            else valueがJSONArrayの場合
                XMLSerializer->>Element: addAttribute("json_class", "array")
                XMLSerializer->>XMLSerializer: processJSONArray(value, element, expandableProperties)
            else valueが基本型の場合
                XMLSerializer->>Element: addAttribute("json_type", 型名)
                XMLSerializer->>Element: appendChild(value.toString())
            end
        end
        
        XMLSerializer->>XMLSerializer: addNameSpaceToElement(element)
        XMLSerializer->>Element: appendChild(element)
    end
    
    Note over XMLSerializer: writeDocumentの処理
    XMLSerializer->>XMLSerializer: new ByteArrayOutputStream()
    XMLSerializer->>XomSerializer: new XomSerializer(baos, encoding)
    XMLSerializer->>XomSerializer: write(doc)
    XMLSerializer->>XMLSerializer: baos.toString(encoding)
    
    XMLSerializer-->>Client: XML文字列
```

8. XMLからJSONへの変換シーケンス
```mermaid
sequenceDiagram
    participant Client as クライアント
    participant XMLSerializer as XMLSerializer
    participant Builder as Builder
    participant Document as Document
    participant Element as Element
    participant JSONObject as JSONObject
    participant JSONArray as JSONArray
    participant JSONNull as JSONNull

    Client->>XMLSerializer: read(xml)
    
    XMLSerializer->>Builder: new Builder()
    XMLSerializer->>Builder: build(new StringReader(xml))
    Builder-->>XMLSerializer: Document
    
    XMLSerializer->>Document: getRootElement()
    Document-->>XMLSerializer: root Element
    
    XMLSerializer->>XMLSerializer: isNullObject(root)
    alt rootがnullObjectの場合
        XMLSerializer-->>Client: JSONNull.getInstance()
    else 処理を続行
        XMLSerializer->>XMLSerializer: getType(root, JSONTypes.STRING)
        XMLSerializer-->>XMLSerializer: defaultType
        
        XMLSerializer->>XMLSerializer: isArray(root, true)
        alt rootが配列の場合
            XMLSerializer->>XMLSerializer: processArrayElement(root, defaultType)
            alt forceTopLevelObjectがtrueの場合
                XMLSerializer->>XMLSerializer: removeNamespacePrefix(root.getQualifiedName())
                XMLSerializer->>JSONObject: new JSONObject().element(key, json)
            end
        else rootがオブジェクトの場合
            XMLSerializer->>XMLSerializer: processObjectElement(root, defaultType)
            alt forceTopLevelObjectがtrueの場合
                XMLSerializer->>XMLSerializer: removeNamespacePrefix(root.getQualifiedName())
                XMLSerializer->>JSONObject: new JSONObject().element(key, json)
            end
        end
    end
    
    Note over XMLSerializer: processObjectElementの処理
    XMLSerializer->>JSONObject: new JSONObject()
    
    alt skipNamespacesがfalseの場合
        loop 各名前空間宣言に対して
            XMLSerializer->>Element: getNamespaceDeclarationCount()
            XMLSerializer->>Element: getNamespacePrefix(j)
            XMLSerializer->>Element: getNamespaceURI(prefix)
            XMLSerializer->>JSONObject: setOrAccumulate("@xmlns" + prefix, uri)
        end
    end
    
    loop 各属性に対して
        XMLSerializer->>Element: getAttribute(i)
        XMLSerializer->>Attribute: getQualifiedName()
        XMLSerializer->>Attribute: getValue()
        alt 型ヒント属性でない場合
            XMLSerializer->>JSONObject: setOrAccumulate("@" + removeNamespacePrefix(attrname), attrvalue)
        end
    end
    
    loop 各子要素に対して
        XMLSerializer->>Element: getChild(i)
        alt childがTextの場合
            XMLSerializer->>Text: getValue()
            alt 値が空白でない場合
                XMLSerializer->>JSONObject: setOrAccumulate("#text", trimSpaceFromValue(text.getValue()))
            end
        else childがElementの場合
            XMLSerializer->>XMLSerializer: setValue(jsonObject, element, defaultType)
        end
    end
    
    Note over XMLSerializer: setValueの処理
    XMLSerializer->>XMLSerializer: getClass(element)
    XMLSerializer->>XMLSerializer: getType(element)
    XMLSerializer->>XMLSerializer: removeNamespacePrefix(element.getQualifiedName())
    
    alt elementが名前空間を持つ場合
        XMLSerializer->>XMLSerializer: processElement(element, type)
        XMLSerializer->>JSONObject: setOrAccumulate(key, simplifyValue(jsonObject, result))
    else elementが属性を持つ場合
        alt elementが関数の場合
            XMLSerializer->>XMLSerializer: 関数処理
            XMLSerializer->>JSONObject: setOrAccumulate(key, new JSONFunction(params, text))
        end
    else 通常の値処理
        alt classが"array"の場合
            XMLSerializer->>XMLSerializer: processArrayElement(element, type)
            XMLSerializer->>JSONObject: setOrAccumulate(key, result)
        else classが"object"の場合
            XMLSerializer->>XMLSerializer: processObjectElement(element, type)
            XMLSerializer->>JSONObject: setOrAccumulate(key, simplifyValue(jsonObject, result))
        else typeに基づく処理
            alt typeが"boolean"の場合
                XMLSerializer->>JSONObject: setOrAccumulate(key, Boolean.valueOf(element.getValue()))
            else typeが"number"の場合
                XMLSerializer->>JSONObject: setOrAccumulate(key, Integer.valueOf(element.getValue()))
            else typeが"string"の場合
                XMLSerializer->>JSONObject: setOrAccumulate(key, trimSpaceFromValue(element.getValue()))
            end
        end
    end
    
    XMLSerializer-->>Client: JSON
```

9.  型ヒント処理の詳細シーケンス
```mermaid
sequenceDiagram
    participant XMLSerializer as XMLSerializer
    participant Element as Element
    participant Attribute as Attribute
    participant JSONObject as JSONObject
    participant JSONArray as JSONArray

    Note over XMLSerializer: JSON → XML 変換時の型ヒント追加
    XMLSerializer->>XMLSerializer: isTypeHintsEnabled()
    alt 型ヒントが有効の場合
        alt valueがbooleanの場合
            XMLSerializer->>Element: addAttribute("json_type", "boolean")
        else valueがnumberの場合
            XMLSerializer->>Element: addAttribute("json_type", "number")
        else valueがstringの場合
            XMLSerializer->>Element: addAttribute("json_type", "string")
        else valueがJSONArrayの場合
            XMLSerializer->>Element: addAttribute("json_class", "array")
        else valueがJSONObjectの場合
            XMLSerializer->>Element: addAttribute("json_class", "object")
        end
    end

    Note over XMLSerializer: XML → JSON 変換時の型ヒント読み取り
    XMLSerializer->>Element: getAttribute("json_type")
    alt 型ヒント属性が存在する場合
        XMLSerializer->>Attribute: getValue()
        alt typeが"boolean"の場合
            XMLSerializer->>JSONObject: Boolean.valueOf(element.getValue())
        else typeが"number"の場合
            XMLSerializer->>JSONObject: Integer.valueOf(element.getValue())
        else typeが"string"の場合
            XMLSerializer->>JSONObject: element.getValue()
        end
    else 型ヒントがない場合
        XMLSerializer->>XMLSerializer: 要素の内容から型を推測
        alt 要素が子要素を持つ場合
            XMLSerializer->>XMLSerializer: オブジェクトとして処理
        else 要素が配列として認識される場合
            XMLSerializer->>XMLSerializer: 配列として処理
        else 要素の値から型を推測
            XMLSerializer->>XMLSerializer: 値の内容から適切な型を決定
        end
    end
```

10. 名前空間処理のシーケンス
```mermaid
sequenceDiagram
    participant XMLSerializer as XMLSerializer
    participant Element as Element
    participant JSONObject as JSONObject
    participant CustomElement as CustomElement

    Note over XMLSerializer: 名前空間の追加処理
    XMLSerializer->>XMLSerializer: addNameSpaceToElement(element)
    
    alt skipNamespacesがfalseの場合
        XMLSerializer->>XMLSerializer: getNamespacesForElement(elementName)
        loop 各名前空間に対して
            XMLSerializer->>Element: addNamespaceDeclaration(prefix, uri)
        end
    end

    Note over XMLSerializer: 名前空間の読み取り処理
    XMLSerializer->>XMLSerializer: hasNamespaces(element)
    alt 要素が名前空間を持つ場合
        XMLSerializer->>Element: getNamespaceDeclarationCount()
        loop 各名前空間宣言に対して
            XMLSerializer->>Element: getNamespacePrefix(j)
            XMLSerializer->>Element: getNamespaceURI(prefix)
            alt prefixが空でない場合
                XMLSerializer->>JSONObject: setOrAccumulate("@xmlns:" + prefix, uri)
            else
                XMLSerializer->>JSONObject: setOrAccumulate("@xmlns", uri)
            end
        end
    end

    Note over XMLSerializer: 名前空間プレフィックスの削除
    XMLSerializer->>XMLSerializer: removeNamespacePrefix(name)
    alt removeNamespacePrefixFromElementsがtrueの場合
        XMLSerializer->>XMLSerializer: name.substring(name.indexOf(":") + 1)
    else
        XMLSerializer->>XMLSerializer: name
    end
```

これらのシーケンス図は、json-lib-2.4の主要な処理フローを示しています：<br>
1.Java→JSON変換: オブジェクトの型に応じた適切な処理とプロセッサーの適用<br>
2.JSON→Java変換: 設定に基づく適切なJavaオブジェクトの生成<br>
3.配列処理: コレクションや配列のJSONArrayへの変換<br>
4.統合処理: JSONSerializerによる統一的な変換インターフェース<br>
5.イベント処理: 処理過程でのイベント通知システム<br>
6.JSON→XML変換: JSONオブジェクトをXMLドキュメントに変換し、型ヒントや名前空間を適切に処理<br>
7.XML→JSON変換: XMLドキュメントをJSONオブジェクトに変換し、型ヒントを読み取って適切な型に変換<br>
8.型ヒント処理: JSONとXML間の型情報の保持と復元<br>
9.名前空間処理: XMLの名前空間をJSONの属性として保持する処理<br>