# 使用 Gson 往 Json 字符串里添加属性

首先是简单的情况，在最外层添加：

```Java
String json = "{\"name\":\"zhangsan\",\"age\":18,\"address\":\"beijing\"}";
JsonObject jsonObj = new Gson().fromJson(json, JsonObject.class);

String greeting = "hello!";
jsonObj.addProperty("greeting", greeting);
System.out.println(new Gson().toJson(jsonObj));

/*
Output:
{"name":"zhangsan","age":18,"address":"beijing","greeting":"hello!"}
*/
```

如果字符串无法解析为 Json，会抛出`JsonSyntaxException`

```Java
try {
    String wrongJson = "aaaaa";
    JsonObject wrongJsonObj = new Gson().fromJson(wrongJson, JsonObject.class);
    wrongJsonObj.addProperty("greeting", greeting);
    System.out.println(new Gson().toJson(wrongJsonObj));
} catch (JsonSyntaxException e) {
    System.out.println("Json Parse Error");
}

/*
Output:
Json Parse Error
*/
```

稍微复杂一点的情况,往深层结构里添加

```Java
People person = new People("zhangsan", 18, "beijing");
person.setPhone(new People.Phone("123", "123456789"));
String json = new Gson().toJson(person);
System.out.println(json);

String greeting = "hello!";

JsonObject jsonObj = new Gson().fromJson(json, JsonObject.class);
jsonObj.getAsJsonObject("phone").addProperty("greeting", greeting);
System.out.println(new Gson().toJson(jsonObj));

// 如果试图取出不存在的JsonObject,会抛出nullPointerExcetion
try {
    jsonObj.getAsJsonObject("inValid").addProperty("greeting", greeting);
} catch (Exception e) {
    System.out.println(e.getMessage());
}

/*
Output:
{"name":"zhangsan","age":18,"address":"beijing",
"phone":{"phoneNumber":"123","phoneType":"123456789"}}
{"name":"zhangsan","age":18,"address":"beijing",
"phone":{"phoneNumber":"123","phoneType":"123456789","greeting":"hello!"}}

Cannot invoke "com.google.gson.JsonObject.addProperty(String, String)" because the return value of "com.google.gson.JsonObject.getAsJsonObject(String)" is null
*/
```
