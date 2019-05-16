---
title: How to programmatically migrate the sequelize instead of sequelize-cli
date: 2019-05-16 22:17:43
tags: ["js", "sequelize"]
categories: ["JavaScript", "Node.js"]
comments: true
---

# 먼저 sequelize-cli 란?

NodeJS로 RDB를 사용하려면 [Sequelize](http://docs.sequelizejs.com/) 처럼 편리한 친구도 없을 것이다. ~~정말?~~

그런 Sequelize를 정말 편리하게 마이그레이션 할 수 있게 도와주는 친구가 하나 있는데, 그게 바로 [sequelize-cli](https://github.com/sequelize/cli) 이다. 사용법은 [링크](http://docs.sequelizejs.com/manual/migrations.html)를 통해 가서 확인해 보자.



# 오늘의 포스팅 주제는?

제목 그대로 sequelize-cli 대신, 코드를 작성해서 Sequelize 를 마이그레이션 하는 방법이다.

이걸 왜 하게 됬냐면, 소스코드 내에 RDB의 암호를 설정파일에 하드코딩 해놨었는데 ~~내가 한건 아니고...~~

보안점검때 딱 걸려서 [AWS KMS](https://aws.amazon.com/ko/kms/) 를 통해 암호를 관리하도록 코드를 수정하게 되었다.

AWS는 정말 없는 서비스가 없는 것 같다... **최고!**

아무튼 그래서 코드 싹 다 고쳐서 서버 구동시 KMS 가서 암호 받아오고 DB connection 맺고 이것 저것 다 초기에 수행하도록 코드 수정 후 배포를 딱 했는데...

배포 스크립트에 아래와 같은 코드가 있었고... 해당 스크립트는 KMS로부터 암호를 받기도 전에 수행되는건 둘째 치고... 암호를 외부에서 주입할 수가 없었다.

```
sequelize db:create
sequelize db:migrate
```

그럼 어떻게? KMS로부터 암호 갖고온 뒤에 마이그레이션을 수행하도록 변경을 해야 했다.

여기까지가 오늘 포스팅을 시작하게 된 이유이다.


# 그럼 어떻게 ~~베꼈어~~?

먼저 [stackoverflow](https://stackoverflow.com/) 에 검색을 해 봤다. ~~다들 google에 검색 먼저 하지만 결국 들어가는데는 여기잖아?~~

제일 처음 들어간 게시물은 [how-to-programmatically-run-sequelize-migrations](https://stackoverflow.com/questions/30848530/how-to-programmatically-run-sequelize-migrations) 인데 `child_process` 를 통해 외부 프로그램 즉 sequelize-cli 를 실행하는 방식이었다.

나쁘진 않지만 내가 사용할 수 없는 방법이었다.

> 나중에 보니 swifty 님이 댓글에 친절하게 umzug 를 언급하고 계셨는데 이제야 봤다...

그러다가 갑자기 든 생각이 *sequelize-cli 코드를 까봐서 베껴보자!* 였다.

일단 github에 가서 코드를 쭉 보니, [아래](https://github.com/sequelize/cli/blob/master/src/commands/migrate.js#L39)가 실제 `db:migrate` 를 수행하는 함수라는 걸 알아냈다. 

```javascript
function migrate(args) {
  return getMigrator('migration', args).then(migrator => {
    return ensureCurrentMetaSchema(migrator)
      .then(() => migrator.pending())
      .then(migrations => {
        const options = {};
        if (migrations.length === 0) {
          helpers.view.log('No migrations were executed, database schema was already up to date.');
          process.exit(0);
        }
        if (args.to) {
          if (migrations.filter(migration => migration.file === args.to).length === 0) {
            helpers.view.log('No migrations were executed, database schema was already up to date.');
            process.exit(0);
          }
          options.to = args.to;
        }
        if (args.from) {
          if (migrations.map(migration => migration.file).lastIndexOf(args.from) === -1) {
            helpers.view.log('No migrations were executed, database schema was already up to date.');
            process.exit(0);
          }
          options.from = args.from;
        }
        return options;
      })
      .then(options => migrator.up(options));
  }).catch(e => helpers.view.error(e));
}
```

그중에 `migrator` 라는 친구를 보니 마지막에 `migrator.up()` 이라는 함수를 호출하는데... 이게 뭔가 하고 `getMigrator` 함수를 쫓아가 보니 [아래](https://github.com/sequelize/cli/blob/master/src/core/migrator.js#L33)와 같은 코드가 있었다.

```javascript
export function getMigrator (type, args) {
  return Bluebird.try(() => {
    if (!(helpers.config.configFileExists() || args.url)) {
      helpers.view.error(
        'Cannot find "' + helpers.config.getConfigFile() +
        '". Have you run "sequelize init"?'
      );
      process.exit(1);
    }

    const sequelize = getSequelizeInstance();
    const migrator = new Umzug({
      storage: helpers.umzug.getStorage(type),
      storageOptions: helpers.umzug.getStorageOptions(type, { sequelize }),
      logging: helpers.view.log,
      migrations: {
        params: [sequelize.getQueryInterface(), Sequelize],
        path: helpers.path.getPath(type),
        pattern: /\.js$/,
        wrap: fun => {
          if (fun.length === 3) {
            return Bluebird.promisify(fun);
          } else {
            return fun;
          }
        }
      }
    });

    return sequelize
      .authenticate()
      .then(() => {
        // Check if this is a PostgreSQL run and if there is a custom schema specified, and if there is, check if it's
        // been created. If not, attempt to create it.
        if (helpers.version.getDialectName() === 'pg') {
          const customSchemaName = helpers.umzug.getSchema('migration');
          if (customSchemaName && customSchemaName !== 'public') {
            return sequelize.createSchema(customSchemaName);
          }
        }

        return Bluebird.resolve();
      })
      .then(() => migrator)
      .catch(e => helpers.view.error(e));
  });
}
```

~~어휴 길다...~~ 바로... [Umzug](https://github.com/sequelize/umzug) 이친구다. `migrator`는 `Umzug`의 인스턴스이자 실질적인 Sequelize의 마이그레이션 프레임워크인 것이었다. *유레카!*

Umzug의 github 사이트를 보면 아래에 너무나도 친절하게 [sequelize-migration-hello](https://github.com/abelnation/sequelize-migration-hello) 를 예제로 알려주고 있다. 얼른 가보자.

# Game Over!

그렇다. 끝났다. [sequelize-migration-hello](https://github.com/abelnation/sequelize-migration-hello) 의 [README.md](https://github.com/abelnation/sequelize-migration-hello/blob/master/README.md)에서 너무나도 친절하게 [migrate.js](https://github.com/abelnation/sequelize-migration-hello/blob/master/migrate.js) 를 통해 마이그레이션을 한다고 알려주고 있었다.

중요한 부분은 아래랑

```javascript
const umzug = new Umzug({
    storage: 'sequelize',
    storageOptions: {
        sequelize: sequelize,
    },

    // see: https://github.com/sequelize/umzug/issues/17
    migrations: {
        params: [
            sequelize.getQueryInterface(), // queryInterface
            sequelize.constructor, // DataTypes
            function() {
                throw new Error('Migration tried to use old style "done" callback. Please upgrade to "umzug" and return a promise instead.');
            }
        ],
        path: './migrations',
        pattern: /\.js$/
    },

    logging: function() {
        console.log.apply(null, arguments);
    },
});
```

아래의 코드다.

```javascript
function cmdMigrate() {
    return umzug.up();
}
```

`Umzug` 에는 `up` 외에 `down` 메소드도 있고, 파라미터로 어디까지 수행할지 정할 수도 있으니 undo, redo, reset 등 다양하게 만들 수 있겠다.

물론 예제에 다 있다.