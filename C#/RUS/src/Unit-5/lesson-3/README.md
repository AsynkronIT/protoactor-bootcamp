# Урок 3: Маршрутиэатор с пулом.

В предыдущем уроке, мы с вами рассматривали, какие типы маршрутизаторов доступны в платформе Proto.Actor. Данные типы маршрутизаторов можно разделить на три разновидности: маршрутизатор, где бизнес логика находится в вашем собственном акторе, маршрутизатор на основе группы акторов и маршрутизатор на основе пула акторов. 

Давайте посветим этот урок маршрутизаторам на основе пула акторов. При использовании пула от вас не требуется создавать маршруты или управлять ими, это будет делать маршрутизатор. Пул можно использовать, когда все маршруты создаются и используются одинаково и нет необходимости специально восстанавливать маршруты. То есть для «простых» маршрутов пул - хороший выбор.

#### Создание маршрутизатора.

Давайте рассмотрим создания маршрутизатором на примере класса BroadcastPool.

```c#
private static readonly Props MyActorProps = Props.FromProducer(() => new MyActor());

var system = new ActorSystem();
var context = new RootContext(system);
var props = context.NewBroadcastPool(MyActorProps, 5);
var pid = context.Spawn(props);
for (var i = 0; i < 10; i++)
{
    context.Send(pid, new Message { Text = $"{i % 4}" });
}
```

Как вы видите, создание маршрутизатора практически ничем не отличается от создания обычного актора.

Прежде всего, мы создаём систему акторов в которой будет находиться наш маршрутизатор, затем мы создаём контекст и с помощью метода контекста `NewBroadcastPool(MyActorProps, 5);` мы создаём экземпляр нашего маршрутизатора.

Метод `NewBroadcastPool(MyActorProps, 5)` принимает Props описывающий наш актор и в качестве второго параметра, количество экземпляров нашего актора.

После того как мы создадим наш маршрутизатор в методе context.Spawn(props); мы сможем оправить ему сообщения с помощью метода 

`context.Send(pid, new Message { Text = $"{i % 4}" });`. 

А он в свою очередь, будет пересылать это сообщение 5 экземплярам нашего актора.

Обычно сообщения, посылаемые маршрутизатору, передаются маршрутам. Но некоторые сообщения маршрутизатор обрабатывает сам. Примером таких сообщений может служить системное сообщение `Stop()` . Данное сообщение будет не пересылаются маршрутам, а будет обработано самим маршрутизатором. В данном случае маршрутизатор будет приостановлен, а поскольку это пул, вместе с ним будут приостановлены и все маршруты, фактически являющиеся его дочерними акторами.

Как мы уже знаем, когда мы посылаем сообщение маршрутизатору, оно будет передано только одному маршруту, по крайней мере, так действует большинство маршрутизаторов. Но есть возможность заставить маршрутизатор разослать сообщение всем маршрутам. Для этого можно использовать ещё одно специальное широковещательное сообщение: RouterBroadcastMessage(). Получив такое сообщение, маршрутизатор перешлёт его содержимое всем маршрутам. Широковещательные сообщения `RouterBroadcastMessage()` можно посылать маршрутизаторам обоих видов - пулам и группам.

#### Удаленные маршруты.

Ранее в этой статье роль маршрутов играли локальные акторы, но надо иметь в виду, что маршрутизаторы способны рассылать сообщения акторам, находящимся на разных серверах. Создать маршрут, действующий на удалённом сервере, совсем не сложно. 

Для этого в платформе Proto.Actor предусмотрены специальные служебное сообщение, которые позволяют добавить или удалить маршрут, в существующий маршрутизатор. Также с помощью этих сообщений при необходимости вы сможете добавить и локальные акторы. Давайте рассмотрим сообщение более подробно.

- RouterAddRoutee - Данное сообшение служит для добавления нового маршрута в маршрутизатор. Оно принимает PID локального или удаленного актора и добавляет его в таблицу маршрутизации.
- RouterRemoveRoutee - В отличие от предыдущего сообщения, данное сообщение удаляет PID актора из таблицы маршрутизации.
- RouterGetRoutees - Для того чтобы, запросить у маршрутизатора, его таблицу маршрутизации применяется данное сообщение.
- Routees - сообшение содержит текушее состояние таблицы маршрутизации нашего роутера.

#### Наблюдение.

Ещё одна функция маршрутизаторов, о которой следует рассказать, это наблюдение. Поскольку маршрутизатор создаёт маршруты, он также является супервизором для этих акторов. 

Маршрутизатор по умолчанию всегда передаёт служебные сообщения своему супервизору. Это может приводить к неожиданным результатам. Когда один из маршрутов потерпит аварию, маршрутизатор сообщит об этом своему супервизору. Этот супервизор почти наверняка захочет перезапустить актор, но вместо маршрута перезапустит маршрутизатор. 

А перезапуск маршрутизатора вызовет перезапуск всех маршрутов, не только аварийного. Со стороны это выглядит так, будто маршрутизатор использует стратегию AllForOneStrategy. Чтобы решить эту проблему, в момент создания маршрутизатора можно настроить свою стратегию.

Тогда когда один из маршрутов потерпит неудачу, только он и будет перезапущен, а все остальные продолжат работать, как ни в чем не бывало. 

В этом уроке вы узнали, насколько гибкими могут быть пулы. К примеру, Вы можете изменять количество маршрутов и даже логику маршрутизации. А когда имеется несколько серверов, можно без ненужных сложностей создавать маршруты на разных серверах. 

Но иногда ограничения, которые накладывают пулы, оказываются слишком строгими, и появляется желание иметь больше свободы в создании и управлении маршрутами. В такой ситуации вам могут пригодиться группы.