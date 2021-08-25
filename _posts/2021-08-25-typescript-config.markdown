---
layout: post
title: "[Typescript] Configuration"
date: 2021-08-24T20:30:00-08:00
---

最近開始使用 Typescript 跟 Nodejs 在新公司的新專案上面，碰上的問題之一就是，該怎麼寫設定檔。

寫設定檔不外乎幾種方式 
 * .env
   ```
   key=value
   ```
 * JSON
   ```json
   {
       "env": "test",
       "database": {
           "host": "db.example.com"
       }
   }
   ```
 * YAML
   ```yaml
   env: test
   # comment
   database:
     host: "db.example.com"
   ```
 * [HOCON](https://github.com/lightbend/config/blob/master/HOCON.md)
   ```
   env: test
   database:
     host: "..."
      database.port: 5678  
   ```

前兩者的缺點很明顯，從 Scala [typesafe config](https://github.com/lightbend/config) (使用 HOCON) 的轉過來我，自然無法接受前三種格式。

.env 的缺點是只支援 key=value 的格式，所有東西都是平的，沒有階層的觀念，也不支援矩陣 ```a,b,c``` 等複雜一點的資料型態。

json 的缺點是，雖然支源了複雜一點的資料型態，但是不能寫註解，跟本不能用啊，專案多人共同進行，怎麼知道為什麼某個欄位在不同的環境為什麼要用不同的值。

yaml 跟 HOCON 沒有上述的缺點，甚至還可以寫 schema 檔，來做格式驗證，但是不知道為什麼在 nodejs 這邊不怎麼流行。

後來想一想，在 *Typescript* 這邊，似乎不用什麼複雜的方案，直接用 Ts 的內建功能就夠了

```typescript
/*
 * This approach supports:
 *  - complex types
 *  - schema validation through type checking.
 *  - comments
 *  - dynamic reloading !!
 */
export type Config = {
    env: "dev" | "stg" | "prod" | "test",
    db: DBConfig
}

export type DBConfig = {
    host: string,
    username: string,
    password: string,
    database: string
}

export const devConfig: Config = {
    env: "dev",
    db: {
        host: process.env.DB_HOST || "localhost",
        username: process.env.DB_USERNAME || "user",
        password: process.env.DB_PASSWORD ||  "p@ssword",
        database: process.env.DB_DATABASE || "db"
    }
}

export function getConfig(env: string = process.env.NODE_ENV || "test"): Config {
    switch (env) {
        case 'test':
            return devConfig;
        case 'dev':
            return devConfig;
        case 'stg':
            return stagingConfig;
        case 'prod':
            return prodConfig;
        default:
            throw new Error(`unsupported environment ${env}`)
    }
}
```
這個方式可以直接寫程式控制設定檔，而且還支援動態載入，修改設定檔，可以透過 nodemon ，不用手動重開 node ，可以自動重新載入新的設定檔，提高工程師的生產力。
