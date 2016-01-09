---
layout: post
title:  "Dependency Injection в Jenkins"
date:   2016-01-10 00:00:00 +0400
author:
    name: Merkushev Kirill
    email: lanwen+blog@yandex.ru
    gravatar: 6ee51971263d8c9a1e70e1dac7418d36
categories: [jenkins]
tags: [jenkins, jenkins plugin]
comments: true
published: true
---

## Куцый механизм со множеством ограничений

Намедни решил попробовать, благо видел в коде ядра ключевые Guice, Injector. Проблемы ощутил почти сразу, причем 
проблемы оказались нерешаемыми на текущем этапе, а разработчики ядра посоветовали этим 
[не пользоваться совсем](https://groups.google.com/forum/#!topic/jenkinsci-dev/ZutbuAfl10s). Но обо всем по порядку.

### Где и как это работает

Для тех, кто не знаком с DI и в частности, [Guice](https://github.com/google/guice), очень советую присмотреться. 
Если совсем коротко, то принцип прост - где то формируется граф зависимостей одних компонентов от других, а сами компоненты 
получают подготовленные зависимости извне - в конструктор или поля в момент инициализации графа. 
Вот как раз библиотека DI и занимается организацией графа и разрешением того кому какую зависимость подсовывать.
 
В Jenkins библиотека для DI появилась не так давно, поэтому глубоко в архитектуру проекта не проникла. 
Основные моменты были уже реализованы без подобных излишеств, поэтому Guice работает стабильно только в одном месте - 
во всем что касается работы с классами, помеченными как `@Extension`. 

#### На практике

Основным и самым распространенным применением может быть замена вызовов вроде `Jenkins.getInstance().getExtensionList(..)`.
Такие вызовы серьёзно затрудняют юнит-тестирование, так как это статический вызов, который непросто заглушить. Кроме того, когда 
таких вызовов в коде достаточно много, читать такой код становится очень некомфортно.

Вместо статического вызова приходит на помощь аннотация `@Inject` повешенная на поле или геттер. 
В момент вызова любого метода вашего расширения, поле будет проинициализировано автоматически. Удобно, правда? 
Пример можно найти например в [GitHub Plugin](https://github.com/jenkinsci/github-plugin/blob/master/src/main/java/org/jenkinsci/plugins/github/config/GitHubPluginConfig.java#L71-L73)

Интереснее со сложным сценарием инжекта. Например, нужно через статический вызов получить 
дескриптор, а дальше через его метод создать объект, который и использовать. В Guice есть возможности вроде `@Provider`. Как их использовать?

Все просто - нужно превратить наш *Extension* в модуль! Для этого всего лишь нужно заимплементить интерфейс `com.google.inject.Module`.
Дальше - все как по мануалу.

### Где это не работает

Это не работает при создании любых не-синглтон объектов. Например Builder, Publisher. Казалось бы - объекты создаются в том же 
месте где и сами *Extension* точки. Но нет, все по-другому. Поэтому в обычных describable придется получать глобальные компоненты 
по-старинке - через статические вызовы.

#### Можно ли с этим что-то сделать? 

Если статических вызовов очень много, можно попытаться сэкономить себе пару строк магическим вызовом:  

{% highlight java %}
Jenkins.getInstance().getInjector().injectMembers(this);
{% endhighlight %}

Это вставит во все поля и геттеры, помеченные как `@Inject` все что зарегистрировано как расширение во внутреннем инжекторе. 

### Где это еще может быть полезно

В тестах! Делаем рулу или метод подготовки:

{% highlight java %}
@Before
public void setUp() throws Exception {
  jRule.getInstance().getInjector().injectMembers(this);
}
{% endhighlight %}

и получаем в полях все нужные объекты без дополнительных усилий. 
Важно только чтобы `JenkinsRule` к этому моменту успешно стартовала.