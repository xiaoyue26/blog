title: Gson与FileUtil/IOUtil笔记
date: 2015-10-05 16:53:51
tags: 
- java
categories:
- java
---

1. 直接从代码学习：
Gson三种基本用法，
```java
Gson gson=new Gson();
Gson gson = new GsonBuilder().registerTypeHierarchyAdapter(Foo.class, new FooAdapter()).create();
Gson gson =new GsonBuilder().registerTypeAdapterFactory(FooAdapterFactory.INSTANCE).create();
```
完整测试代码如下：(其中TypeAdapterFactory的测试来自Stackoverflow,作者还用上了单例)
`Foo.java`
```java
package testGson;

import java.io.Serializable;

public class Foo implements Serializable {
    private static final long serialVersionUID = -1455994233292753460L;

    public Foo() {
        setDataString("data");
        setIdInteger(100);
        setNameString("name");
        setValueString("value");
    }

    public String getNameString() {
        return nameString;
    }

    public void setNameString(String nameString) {
        this.nameString = nameString;
    }

    public String getValueString() {
        return valueString;
    }

    public void setValueString(String valueString) {
        this.valueString = valueString;
    }

    public Integer getIdInteger() {
        return idInteger;
    }

    public void setIdInteger(Integer idInteger) {
        this.idInteger = idInteger;
    }

    public String getDataString() {
        return dataString;
    }

    public void setDataString(String dataString) {
        this.dataString = dataString;
    }

    String nameString;
    String valueString;
    Integer idInteger;
    String dataString;
}

```

---

`FooAdapter.java`
```java
package testGson;

import java.io.IOException;

import com.google.gson.TypeAdapter;
import com.google.gson.stream.JsonReader;
import com.google.gson.stream.JsonWriter;

public class FooAdapter extends TypeAdapter<Foo>{

    @Override
    public void write(JsonWriter out, Foo obj) throws IOException {
        try {
            out.beginObject();
            out.name("idIntegerFromAdapter").value(obj.getIdInteger());
            out.name("nameStringFromAdapter").value(obj.getNameString());
            out.name("valueStringFromAdapter").value(obj.getValueString());
            out.name("dataStringFromAdapter").value(obj.getDataString());
            out.endObject();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @Override
    public Foo read(JsonReader in) throws IOException {
        final Foo obj = new Foo();
        in.beginObject();
        while (in.hasNext()) {
            switch (in.nextName()) {
                case "idIntegerFromAdapter":
                    obj.setIdInteger(in.nextInt());
                    break;
                case "nameStringFromAdapter":
                    obj.setNameString(in.nextString());
                    break;
                case "valueStringFromAdapter":
                    obj.setValueString(in.nextString());
                    break;
                case "dataStringFromAdapter":
                    obj.setDataString(in.nextString());
                    break;
                default:
                    break;
        
            }
        }
        in.endObject();
        return obj;
     }
}

```
---
`FooAdapterFactory.java`
```java
package testGson;

import java.io.IOException;

import com.google.gson.Gson;
import com.google.gson.TypeAdapter;
import com.google.gson.TypeAdapterFactory;
import com.google.gson.reflect.TypeToken;
import com.google.gson.stream.JsonReader;
import com.google.gson.stream.JsonWriter;


public enum FooAdapterFactory implements TypeAdapterFactory {
    INSTANCE; // Josh Bloch's Enum singleton pattern

    @SuppressWarnings("unchecked")
    @Override
    public <T> TypeAdapter<T> create(Gson gson, TypeToken<T> type) {
        if (!Foo.class.isAssignableFrom(type.getRawType())) return null;
        // Note: You have access to the `gson` object here; you can access other deserializers using gson.getAdapter and pass them into your constructor
        return (TypeAdapter<T>) new FooAdapter();
    }

    private static class FooAdapter extends TypeAdapter<Foo> {
        @Override
        public void write(JsonWriter out, Foo obj) {
            try {
                out.beginObject();
                out.name("IdInteger").value(obj.getIdInteger());
                out.name("NameString").value(obj.getNameString());
                out.name("ValueString").value(obj.getValueString());
                out.name("DataString").value(obj.getDataString());
                out.endObject();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }

        @Override
        public Foo read(JsonReader in) throws IOException {
            final Foo obj = new Foo();
            in.beginObject();
            while (in.hasNext()) {
                switch (in.nextName()) {
                    case "IdInteger":
                        obj.setIdInteger(in.nextInt());
                        break;
                    case "NameString":
                        obj.setNameString(in.nextString());
                        break;
                    case "ValueString":
                        obj.setValueString(in.nextString());
                        break;
                    case "DataString":
                        obj.setDataString(in.nextString());
                        break;
                    default:
                        break;
            
                }
            }
            in.endObject();
            return obj;
         }
    }
}
```
---
`FactoryTest.java`
```java
package testGson;

import java.io.File;
import java.io.IOException;

import static java.lang.System.out;

import org.apache.commons.io.FileUtils;

import com.google.gson.Gson;
import com.google.gson.GsonBuilder;

public class FactoryTest {

    public static void main(String[] args) {
        Foo foo = new Foo();
        /**
         * 1. simple gson:
         * */
       /* Gson gson = new Gson();*/
        /**
         * 2. gson with TypeAdapter:
         * */
        Gson gson = new GsonBuilder().registerTypeHierarchyAdapter(Foo.class, new
                  FooAdapter()) .create();
        /**
         * 3. gson with TypeAdapterFactory:
         * */
       /* Gson gson =new GsonBuilder().registerTypeAdapterFactory(FooAdapterFactory.INSTANCE).create();
        */
        String jsonString = gson.toJson(foo);
        File tempStore = new File("input/tempStore.json");
        try {
            FileUtils.writeStringToFile(tempStore, jsonString);
        } catch (IOException e) {
            e.printStackTrace();
        }
        Foo foo2=gson.fromJson(jsonString, Foo.class);
        out.println(foo2.dataString);
        out.println(foo2.idInteger);
        out.println(foo2.nameString);
        out.println(foo2.valueString);
    }

}

```
---
`maven pom.xml`
```xml
<dependency>
			<groupId>commons-io</groupId>
			<artifactId>commons-io</artifactId>
			<version>2.4</version>
			<!--FileUtil和IOUtil-->
</dependency>
<dependency>
			<groupId>com.google.code.gson</groupId>
			<artifactId>gson</artifactId>
			<version>2.3</version> 
</dependency>
```
`ParseTokenExample.java`
```java
package testGson;

import java.io.File;
import java.io.IOException;
import java.io.StringReader;
import java.net.MalformedURLException;
import java.net.URL;
 


import org.apache.commons.io.FileUtils;
import org.apache.commons.io.IOUtils;
 


import com.google.gson.stream.JsonReader;
import com.google.gson.stream.JsonToken;
 
public class ParseTokenExample{
    public static void main(String[] args) throws MalformedURLException, IOException {
        String url = "http://freemusicarchive.org/api/get/albums.json?api_key=60BLHNQCAOUFPIBZ&limit=5";
        String json = IOUtils.toString(new URL(url));
        //使用reader去读取json
        JsonReader reader = new JsonReader(new StringReader(json));
        File file = new File("input/jsondata.json");
        FileUtils.writeStringToFile(file, json);
       // handleObject(reader);
    }
    private static void handleObject(JsonReader reader) throws IOException {
        reader.beginObject();
        while (reader.hasNext()) {
            JsonToken token = reader.peek();
            if (token.equals(JsonToken.BEGIN_ARRAY))
                handleArray(reader);
            else if (token.equals(JsonToken.END_ARRAY)) {
                reader.endObject();
                return;
            } else
                handleNonArrayToken(reader, token);
        }
 
    }
    /**
　　　*处理json数组，第一个标记是 JsonToken_BEGIN_ARRAY,数组可能包含对象或者基本原始类型
     *
     * @param reader
     * @throws IOException
     */
    public static void handleArray(JsonReader reader) throws IOException {
        reader.beginArray();
        while (true) {
            JsonToken token = reader.peek();
            if (token.equals(JsonToken.END_ARRAY)) {
                reader.endArray();
                break;
            } else if (token.equals(JsonToken.BEGIN_OBJECT)) {
                handleObject(reader);
            } else
                handleNonArrayToken(reader, token);
        }
    }
 
    /**
     * 处理不是数组的符号标记
     *
     * @param reader
     * @param token
     * @throws IOException
     */
    public static void handleNonArrayToken(JsonReader reader, JsonToken token) throws IOException {
        if (token.equals(JsonToken.NAME))
            System.out.println(reader.nextName());
        else if (token.equals(JsonToken.STRING))
            System.out.println(reader.nextString());
        else if (token.equals(JsonToken.NUMBER))
            System.out.println(reader.nextDouble());
        else
            reader.skipValue();
    }
}
```

2. 学习gson的adapter(看懂了上面代码就不用看这个了)：
http://www.javacreed.com/gson-typeadapter-example/
 

3.进阶学习(嵌套内部类，复杂类)：
http://www.cnblogs.com/zhangminghui/p/4109539.html

个人想法:
最根本的办法是减少耦合，减少依赖关系，尽量存储id,而不是对象引用，这样能最大程度地减少复杂代码。
