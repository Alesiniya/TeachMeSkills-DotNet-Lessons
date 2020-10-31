# Запуск миграций EF Core из вашего кода

Все пользователи Entity Framework/EF Core знают, что запустить миграции из консоли или менеджера пакетов можно с помощью команды:

```
Update-Database
```

Кроме того, можно использовать команду dotnet ef:

```
dotnet ef database update
```

Но иногда может потребоваться запускать миграции по запросу из кода C#, например, вызвав конечную точку API из панели администратора сайта. Как правило, это было проблемой в более старых версиях Entity Framework, когда «версия» базы данных не совпадала с «версией» нужной EF Code. В этом случае «всё падало». В EF Core с этим меньше проблем, так как ваш код не рухнет, пока не попытается сделать что-то, что не может завершить (например, выбрать столбец, который ещё не существует). Но все же есть случаи, когда нужно развернуть код, протестировать его работу в промежуточной среде с действующей базой данных, а затем запустить миграцию базы данных по запросу.

## Миграция EF Core в C#

На самом деле все очень просто.

```
var migrator = _context.Database.GetService<IMigrator>();
await migrator.MigrateAsync();
```

Где \_context – это контекст вашей базы данных.

## Проверка ожидающих миграций

Также может быть нужно проверить, есть ли ожидающие миграции, прежде чем пытаться их запустить. Например, может быть полезно узнать, в каком состоянии находится база данных, из панели администратора:

```
await _context.Database.GetPendingMigrationsAsync();
```

## Миграция при запуске приложения

В некоторых случаях вам всё равно, когда выполняется миграция, вы просто хотите, перенести базу данных до запуска приложения. Это подойдёт для проектов, в которых время миграции базы данных не имеет значения или оно относительно небольшое. Например, для мало нагруженных веб-сайтов, расположенных на одном сервере. Для этого в .NET Core есть парадигма «Фильтров Запуска» (StartupFilter). Код будет примерно следующим:

```
public class MigrationStartupFilter<TContext> :
  IStartupFilter where TContext : DbContext {
  public Action<IApplicationBuilder>
    Configure(Action<IApplicationBuilder> next) {
   return app => {
    using (var scope =
     app.ApplicationServices.CreateScope()) {
    foreach (var context in
      scope.ServiceProvider.GetServices<TContext>()) {
     context.Database.SetCommandTimeout(160);
     context.Database.Migrate();
    }
   }
   next(app);
  };
 }
}
```

Фильтры запуска в .NET Core в основном похожи на фильтры в MVC. Они встраиваются в процесс запуска и выполняют код перед запуском приложения, причём только при запуске. Если вы когда-либо использовали Application_Start() в global.asax в .NET Framework, то это очень похоже.

Осталось добавить наш фильтр в контейнер DI (https://t.me/NetDeveloperDiary/748) в Startup.cs:

```
services.AddTransient<IStartupFilter,
  MigrationStartupFilter<Context>>();
```

Здесь Context - это контекст нашей базы данных.