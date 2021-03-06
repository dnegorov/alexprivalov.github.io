---
layout: post
title: "Использование Docker для кросскомпиляции проектов"
permalink: docker-cpp
tags: C++ Docker GitLab
---

<img src='/assets/docker-c++/cplus-docker.jpg' style="width: 720px;">
Docker как незаметный и незаменимый центральный элемент в разработке, Continuous Integration под множество разных целевых платформ.
Как не проходить квест по настройке окружения каждый раз.

---

**Важно**: все нижесказаное описывает мой опыт работы в основном с embedded linux C/C++ проектами. Просьба помнить это во время чтения.

## Содержание
- [Проблема](#problem)
- [Постановка задачи](#formulation)
- [Решение](#solution)
    - [Docker](#docker)
    - [Docker Registry](#registry)
    - [Скрипт-обертка для запуска docker образов](#wrapper_script)
    - [Автоматическая сборка образов](#docker_autobuilds)
    - [Контейнеры и Continuous Integration(на базе GitLab)](#docker_and_ci)
    - [Минусы и их решение](#docker_minuses)
- [Заключение](#resume)

## Проблема    {#problem}
Сколько времени должен потратить новый сотрудник на то, чтобы собрать и запустить незнакомый ему проект? Обычно это суммарное время, необходимое на следующие действия:

 - скачивание/клонирование репозитория проекта;
 - настройка окружения разработчика: 
    - установка/настройка средств разработки(toolchains, IDE, libs);
    - разложение всего по правильным путям, конфигурирование make/cmake/qmake-файл/project_build.sh для сборки;
 - саму сборку;
 - запуск/деплой приложения/сервиса;

По дороге может выясниться, что где-то чего-то не хватает в зависимостях, неправильно прописан путь к либе/утилите/тулчейну, не та версия библиотеки установлена, где-то не хватает симлинка, где-то лажа с правами. И пока решишь все эти мелкие проблемы - пройдет значительный кусок времени. Плюс еще по любому будут привлекаться сотрудники, которые уже давно работают с проектом. А те, в свою очередь, со скрипом вспоминают в чем был вопрос, так как проблема была решена полгода назад и о ней уже успели забыть.

Обычно для того, чтобы каждый раз не набивать шишки, пишется некий README/вики-статья, где прописываются **точные** инструкции, по выполнению которых можно добиться воспроизводимого результата. За пару человек эти инструкции отлаживаются, и наступает счастье.

В идеале хочется, чтобы сборка проекта выглядела подобным образом:  

{% highlight bash %}
    install all required software, SDKs, libs
    git clone git@project.git
    cd project_dir
    make/cmake/build_project.sh
    deploy
    run
{% endhighlight %}

Какие варианты решений я встречал/проходил:

 - каждый сам себе все настраивает: долго, не воспроизводимо(проблема ["у меня на компьютере все работает"](http://lurkmore.to/%D0%A3%D0%9C%D0%92%D0%A0), так как у каждого разработчика разное окружение), подвержено ошибкам конфигураций. При выпуске новой версии SDK - прохождение квеста каждым разработчиком заново. Автоматизация сборки окружения shell скриптом может частично решить проблему;
 - единое окружение у всех разработчиков, варианты:
    - единожды настроенная виртуальная машина билд-мастером, все ее себе копируют и работают локально. Новое SDK/библиотека/что-то пропатчено/какое-то важное изменение - новая виртуалка. Тяжело, много места, неудобно;
    - настроенные сервера, у каждого есть свой аккаунт, все туда логинятся по ssh, у каждого своя копия исходников, которая примонтирована на локальную машину разработчика по samba/nfs/sshfs/whatever, и все собираются в уже готовом и настроенном окружении. Сеть/сервер упали - разработчики ковыряются в носу. Качество работы определяется качеством работы сети, в целом невысокая производительность работы IDE(парсинг проекта, подсветка), так как сеть в любом случае проигрывает в скорости доступа локальному диску. Очень осторожно надо менять настройки окружения, ибо если накосячить случайно - зацепит всех, не классно;
    - [linux контейнеры](https://ru.wikipedia.org/wiki/Виртуализация_на_уровне_операционной_системы) - об этом и поговорим далее.

## Постановка задачи    {#formulation}

Представим следующую ситуацию:

 - есть n-ное количество целевых платформ(разные версии linux, gcc, разные процы: [ARM](https://ru.wikipedia.org/wiki/ARM_(%D0%B0%D1%80%D1%85%D0%B8%D1%82%D0%B5%D0%BA%D1%82%D1%83%D1%80%D0%B0)), [MIPS](https://ru.wikipedia.org/wiki/MIPS_(%D0%B0%D1%80%D1%85%D0%B8%D1%82%D0%B5%D0%BA%D1%82%D1%83%D1%80%D0%B0)), [SuperH](https://ru.wikipedia.org/wiki/SuperH), x86-64), под которые проекты собираются, соответственно, разные тулчейны для кросскомпиляции;
 - под каждый конкретный процессор идет собственное SDK от чипмейкера. Есть различные версии одного и того же SDK;
 - в SDK находятся баги, их патчат(причем иногда это заметно "сверху", т.е. пользователям SDK, так как могут поменяться интерфейсы), соответственно SDK необходимо пересобрать, бинари/хедеры расшарить между разработчиками;
 - то же самое касается еще ряда библиотек, которые работают поверх SDK, например, Qt, Chromium;
 - необходимо свести к минимуму время на развертывание окружения для разработки под конкретную платформу;
 - большинству разработчиков вообще не надо или не интересно заморачиваться настройкой/обновлением окружения, им бы сразу попасть в готовое, где можно писать уже непосредственно код;
 - весь зоопарк платформ надо как-то тестировать, желательно автоматически, хотя бы на предмет успешной компиляции кода проекта под все платформы, и при этом не хочется постоянно перенастраивать CI для того, чтобы поддерживать окружения в актуальном состоянии;

Эволюционно решение предстало в виде контейнеризации окружения. Одна платформа - один контейнер со всем необходимым содержимым. Каждый разработчик может легко скопировать его себе на машину и тут же начать работать. Это несомненно проще, чем проходить квест по настройке/обновлению окружения в соответствии с инструкцией.

Сразу же на поверхность выплыл еще один вопрос: распространение контейнеров. Бегать с флешкой/ходить по сети куда-то - не классно, плюс человеческая натура такова, что даже если десять раз скомандовать в рабочем чатике: "обновляемся!", - момент действия будет отложен как можно дальше. Возможность централизованной раздачи контейнеров, да еще если это можно сделать автоматически, "незаметно" для разработчика - это будет просто песня!

Из всех вариантов(LXC, [OpenVZ](https://openvz.org/), [Docker](https://www.docker.com/) и т.д.), которые были доступны на момент исследований, лишь Docker с его инфраструктурой подошел целиком и полностью.

## Решение  {#solution}

### Docker  {#docker}

Цитата с [вики](https://ru.wikipedia.org/wiki/Docker):

    Docker — программное обеспечение для автоматизации развёртывания и управления приложениями в среде виртуализации на уровне операционной системы. Позволяет «упаковать» приложение со всем его окружением и зависимостями в контейнер, который может быть перенесён на любую Linux-систему с поддержкой cgroups в ядре, а также предоставляет среду по управлению контейнерами.

Как было до внедрения docker'a: были серверы, где располагались настроенные/собранные sdk/библиотеки под конкретные таргеты. Все заинтересованные разработчики логинились туда по ssh, монтировали с сервера на свой локальный хост по nfs/samba/sshfs папки с проектами, открывали их в локальном любимом редакторе. На локальном хосте редактировали, на удаленном сервере - компилировали. Это работало: у всех единое окружение, настраивать его всем каждый раз не надо, для нового разработчика просто заводим новый аккаунт на сервере. Но медленно: проекты большие, IDE постоянно подвисает на сетевых операциях, на сервере всегда кто-то компилирует - ты ждешь пока скомпилятся твои изменения, в то время как твой хост с 4хядерным Intel Core i7 и 16 Gb ОЗУ просто простаивает, печаль:(.

После введения контейнеров, как у нас в компании получилось:

 - каждый разработчик работает локально, не зависит от сети и ее лагов;
 - под одну платформу/таргет есть две "последние" версии docker-образа: latest и latest_debug. Первая версия - это почищенная вторая от временных файлов(обьектники, архивы, сырцы либ, .git-папки и все то, что точно не потребуется для разработки приложений, просто использующих sdk/библиотеки). Соответственно, latest_debug - это просто дамп всего того, что нагенерировалось во время сборки sdk и библиотек;
 - разработчики приложений - используют готовое запакованное в latest docker-образ окружение для сборки проектов;
 - разработчики платформы - пилят и патчат саму программную "платформу": то что потом используется разработчиками приложений. Т.е. linux, SDK от чипмейкера, Qt, Webkit\Chromium, Busybox, etc. Для этого используют latest_debug версию контейнера. Не страшно накосячить и что-то не так сделать - в крайнем случае все просто потеряется и откатится к первоначальному состоянию. Потому можно смело экспериментировать, не боясь "сломать" достаточно сложную многоступенчатую сборку "мира", которая уже настроена. Опять же, удобно использовать latest_debug версию из-за того, что уже есть все в собранном состоянии, при изменениях компилируется только дельта. Т.е. не приходится ждать, пока соберется, например, весь Qt5 Framework(который еще надо и корректно сконфигурировать для кросскомпиляции!);
 - latest docker-образы используются на CI. Всегда есть платформы в состоянии активной разработки и те, которые сейчас на поддержке: "туда разработчики редко заходят". До введения CI часто возникали ситуации, когда билд был сломан для редкоразрабатываемой на данный момент платформы. И прежде чем приступить к работе на ней, необходимо было в обязательном порядке фиксить ошибки сборки. CI просто исключил такие ситуации - виновник <s>торжества</s> поломки сразу получает письмо от GitLab и бежит исправлять ошибки;
 - введен в строй приватный [docker registry](#registry) - сервер хранения docker образов - все заинтересованные получают образы только с этого сервера. Туда же выкладываются обновленные версии образов;


### Docker Registry {#registry}

С офф. [сайта](https://docs.docker.com/registry/):

    The Registry is a stateless, highly scalable server side application that stores and lets you distribute Docker images. The Registry is open-source, under the permissive Apache license.
    
В общем-то все сказано:). Централизованное хранение и раздача docker-образов. Можно скидывать образы на какой-то ftp/web-сервер, облако. При таком варианте каждый разработчик должен сам сознательно скачать на свою машину необходимый образ; что-то вроде следующих действий:

    wget https://cloud.company.com/dockers/specific_img_version.tar
    docker load specific_img_version.tar
    rm -f specific_img_version.tar

Где specific_img_version.tar - название экспортированного docker образа.

А можно поднять свой Docker Registry, откуда сам docker клиент будет тянуть требуемый образ.

В общем случае мне кажется логичным использовать для этих целей [Docker Hub](https://hub.docker.com/) - аналог GitHub в мире docker контейнеров. Там можно размещать неограниченное количество публичных образов и только один приватный. Большее количество приватных образов можно добавлять за отдельную плату. Если нет ничего секретного - заводим аккаунт, выкладываем образы, которые публичны и доступны всем. Если хотим "приватности" - покупается [подписка](https://hub.docker.com/billing-plans/) и можно размещать приватные образы, т.е. публично не доступные. Эти два варианта клевые тем, что сразу снимается вопрос поддержки - за нас это делают админы Docker Hub. В нашем случае это не подходило(политика компании, есть компоненты от третьих сторон, которые мы не имеем права хранить за пределами компании), потому подняли собственный docker registry. Он не такой крутой, как Docker Hub. Нет, например, какого либо веб-интефейса. "Общаться" с сервером можно через rest api. Например, вот как получить список репозиториев при помощи curl'a:
    
{% gist 49ff2e587ed6289b3806379e3d95d8a5 docker_registry_rest_api %}

Сам сервер поставляется в виде, что не удивительно, docker-образа и запускается следующим образом:

{% gist 5c1c2c57cd0b33feecf32385a414fdf8 run_docker_registry.sh %}
     
где:

 - **\-\-restart=always** - всегда перезапускать контейнер, если по какой-то причине контейнер был остановлен(упал процесс внутри, хост-система перезагрузилась);
 - **/docker_registry** - папка на сервере, которая будет хранить все выгруженные в registry образы, "пробрасывается" внутрь контейнера по пути /var/lib/registry (конструкция **-v /docker_registry:/var/lib/registry**);
 - **REGISTRY_HTTP_TLS_CERTIFICATE**, **REGISTRY_HTTP_TLS_KEY** - переменные окружения в контейнере, которые содержат пути к сертификату и ключу;
 - более подробное [руководство](https://docs.docker.com/registry/deploying/) по registryу:)
 - сами сертификаты не хранятся внутри контейнера, а также "пробрасываются" внутрь него: **-v /docker_registry/certs:/certs**;
    
После выполнения вышеуказанной команды, у вас есть настроенный и готовый для работы registry.

**Важное примечание**: docker client по умолчанию работает через https. Если у вас нет/не планируется SSL для домена, на котором будет крутиться registry, то на каждом хосте, где будет запускаться docker client, необходимо [прописать](https://docs.docker.com/registry/insecure/#deploy-a-plain-http-registry) адрес "небезопасного" docker registry в /etc/docker/daemon.json:

    {
      "insecure-registries" : ["registry_ip_without_ssl.com:5000"]
    }

По умолчанию docker client всегда "смотрит" на Docker Hub. Для того чтобы он мог пушить и пулить образы с других хранилищ, в имя образа всегда надо добавлять адрес docker registry. Пример имени, который используется у нас в компании:

    hub.company.com:5000/chipmaker_name/platform_name:latest

где hub.company.com:5000 - url и порт, на котором работает docker registry, если эта часть пустая, то запросы будут уходить на Docker Hub.
Вся дальнейшая часть имени может быть абсолютно произвольной.

Отправить образ в хранилище:

    docker push registry_ip:registry_port/image_name:image_version

Получить образ из хранилища:

    docker pull registry_ip:registry_port/image_name:image_version

В общем-то, это все команды, которые используются у нас в компании для взаимодействия между docker client и docker registry.


Схема работы такова:

 - на рабочем компьютере билдмастера(либо на CI) собирается образ, который содержит все необходимое для разработки под конкретную платформу, проверяется его валидность и отправляется в registry:
{% highlight bash %}
docker push registry_ip:registry_port/image_name:image_version
{% endhighlight %}
 - все желающие забирают/пулят образ из registryа и работают в готовом окружении:
{% highlight bash %}
docker pull registry_ip:registry_port/image_name:image_version
{% endhighlight %}
 - удаляют локальную копию предыдущей версии образа, **по хешу(IMAGE ID), а не по имени**:
{% highlight bash %}
docker rmi e90ae3da554e
{% endhighlight %}

Можно, конечно, и по имени:тегу, но у одного образа может быть несколько имен, а хеш - он уникальный.

Запуск:
{% highlight bash %}
docker run -it --rm -v /path/to/project/sources:/path/of/sources/inside/container registry_ip:registry_port/image_name:image_version bash
{% endhighlight %}
В результате получаем обычную командную строку(в данном случае bash), в которой можно перейти в папку с проектом и вызвать make/cmake/build.sh
Можно запустить сразу билд, чтобы ручками не ходить куда-то:

{% highlight bash %}
docker run -it --rm -v /path/to/project/sources:/path/of/sources/inside/container registry_ip:registry_port/image_name:image_version bash -c "cd /path/of/sources/inside/container; make/cmake/build.sh;"
{% endhighlight %}
Команду выше можно использовать для интеграции с IDE: в каждой приличной IDE есть что-то вроде custom build steps, куда приведенную команду можно прописать.

Все вышеуказанные команды имеют право на жизнь и использование, но ... большинству разработчиков они не нужны/интересны. Потому вокруг докера была написана простая обертка на bash.

### Скрипт-обертка для запуска docker образов {#wrapper_script}

 - одна и та же везде: и для разработчиков, и для CI;
 - лежит в git вместе с проектом;
 - решает следующие задачи:
    - нулевое вхождение: не надо изучать docker, чтобы пользоваться;
    - знает, откуда и какой актуальный образ окружения для нужной платформы спулить;
    - создает и запускает новый контейнер;
    - "пробрасывает" папку с проектом внутрь контейнера;
    - создает "на лету" пользователя, аналогичного хостовому - нет проблем с правами на сгенерированные файлы(те, которые получились в результате компиляции, так как в контейнере по умолчанию процессы работают от рута);
    - настраивает корректную работу git внутри контейнера;
    - проверяет на наличие и удаляет "висящие" слои/образы, остановленные контейнеры - место на диске не пропадает впустую;
    - кого не устраивает дефолтная функциональность - используют докер напрямую с его километровыми командами:)

скрипт запуска:
{% gist 3624e6183393dbbb6f7ee251e1e04f8d docker_wrapper.sh %}

Как используем скрипт:
{% highlight bash %}
cd project_dir
./dockerBuild.sh platformName mode path_to_project optional_cmd
{% endhighlight %}
где:

 - dockerBuild.sh - скрипт-обертка над докером;
 - platformName - имя платформы, под которую необходимо собрать проект;
 - mode - режим работы; их всего два: dev и CI. Первый - режим разработчика: получаем bash c [cwd](https://en.wikipedia.org/wiki/Working_directory) в папке с проектом, где можно собрать проект, работать с гитом и т.п. Второй режим - запускает автоматически сборку, используется на CI;
 - path_to_project - очевидно, путь к проекту, в данном случае “.”(точка, текущий путь);
 - optional_cmd - необязательный параметр, имеет смысл только в dev-режиме: команда, которую выполнить в контейнере и выйти.

### Контейнеры и Continuous Integration(на базе GitLab) {#docker_and_ci}

Легкость разворачивания окружения на любом линуксе "спровоцировала" логичное продолжение в виде Continuous Integration. В строю был GitLab, Docker Registry. Для полноценного счастья не хватало [GitLab Runner](https://docs.gitlab.com/runner/)'a, на котором собственно будет происходить сборка проекта под разные платформы в окружениях, которые будут вытягиваться с Docker Registry:
![](/assets/docker-c++/slide1.png)
<span class="signed-image">Что происходит по push'у разработчика в репозиторий проекта</span>

Спустя время, на настроенный Runner была навешена еще функция еженочных тестов на текущем срезе кода.

### Автоматическая сборка образов {#docker_autobuilds}

Спустя некоторое время, после того как использование docker образов было поставленно на поток, сборка новых версий стала отнимать много времени. Каждый раз необходимо было делать одни и те же действия по выпуску нового окружения. Эволюционно пришли к мысли: нужен CI для окружений!

Итого:
 - есть репозиторий, который содержит [dockerfile](https://docs.docker.com/engine/reference/builder/)'ы, bash-скрипты, конфиги для сборки docker-образа под ту или иную платформу;
 - есть репозиторий с патчами для SDK, структура приблизительно следующая:
 
{% highlight bash %}
Patches_repo  
    |-->Platform1  
    |       |-->SDKv1  
    |       |     |-->01_patch_name.patch  
    |       |     |-->02_patch_name.patch  
    |       |     |-->.....  
    |       |     |-->nn_patch_name.patch  
    |       |-->SDKv1.5  
    |             |-->01_patch_name.patch  
    |             |-->02_patch_name.patch  
    |             |-->.....  
    |             |-->nn_patch_name.patch  
    |-->Platform2  
    |       |-->SDKv1  
    |       |     |-->...  
    |       |-->SDKv1.7  
    |             |-->...  
    |-->....  
    |-->PlatformN  
            |-->...  
{% endhighlight %}
 
 - есть сами оригинальные SDK от чипмейкеров;
 - любые изменения в репозитории с патчами - сигнал к тому что необходимо обновить образ - у всех должно быть последнее актуальное SDK;
 - на push в репозиторий с патчами настроен [pipeline](https://docs.gitlab.com/ee/ci/pipelines.html), который каждый раз запускает утилиту patch в контейнере с чистыми sdk - успешно ли все патчится? не ноль в качестве возврата программы patch - тот, кто сделал изменения получает письмо с <s>угрозами</s> ошибками и должен исправить ошибку;
 - на репозиторий с патчами настроен [scheduled pipeline](https://docs.gitlab.com/ce/user/project/pipelines/schedules.html): в полночь запускается скрипт, который анализирует, что поменялось относительно предыдущего билда. Если есть изменения - собирает образ(ы) под соответствующую платформу, валидирует образ(успешно ли собирается проект в контейнере? запускается ли он на таргете?), push'ит результат в docker registry и шлет письмо-уведомление на почту;
 - с утра разработчики начинают работать с кодом, запускают dockerBuild.sh, чтобы скомпилить проект, тот в свою очередь pull'ит новособранный образ, локально удаляет старый.
    
![](/assets/docker-c++/slide2.png)
<span class="signed-image">Как собирается docker образ</span>
    
## Минусы и их решение {#docker_minuses}

 - по умолчанию в докер-контейнере процессы работают с рутовыми правами. Из-за этого возникают проблемы с правами: файлы, сгенеренные рутовым пользователем в контейнере на хосте может менять только рут. Неудобно. Решение: при запуске контейнера на лету создается пользователь с id, равным id пользователя хоста, который запустил скрипт-обертку;
 - плохая интеграция с IDE, частично решено:
    - запуск компиляции осуществляется через скрипт-обертку. Вызов этого скрипта легко прописывается в настройки проекта;
    - IDE не имеет доступа к содержимому контейнера, так как контейнер вещь в себе. Потому парсинг проекта не работает - весь SDK с хедерыми не доступен. Полукостыльное решение - необходимые хедеры просто скопированы на хост. Зачастую этого достаточно; 
 - быстро кушающееся место на диске при достаточно частом обновлении образов, решилось парой команд в скриптах запуска контейнеров:

{% gist 506f6084edf32895efaf765e523e652d force_clean_dangling_layers.sh %}

 - сложно понять различия между двумя образами для одной платформы, но с разными датами сборки. Частично решено это введением паспорта версии, которая ложится внутрь образа: когда собран, хеши гитов, еще что-то. Решение в лоб: копируется содержимое на хост обоих образов и сравнивается компаратором(diff, meld): много места, тяжело, <s>бессмысленно</s> стопроцентно;
 - администратору docker registry неудобно его "администрировать". Нет web ui. Казалось бы, типичная задача: просмотреть список доступных репозиториев и образов в них, только через rest api. Или удалить с registry устаревший образ, дабы освободить место - этого даже нет в rest api. Без хаков просто не обойтись. Решение на будущее: у GitLab есть [встроенный](https://docs.gitlab.com/ce/user/project/container_registry.html) docker registry, со всеми красивостями и удобствами, возможно стоит мигрировать на него;
 - у меня не получилось с наскока сделать в Docker Registry разграничение по правам доступа. Типичный сценарий: анонимный pull и авторизованный push. Соответствующее [обсуждение](https://github.com/docker/distribution/issues/1028) на гитхабе. Когда я интересовался вопросом, то было две опции:
    - HTTP basic авторизация - все авторизированные пользователи имеют равные права, не то что надо;
    - авторизация через [nginx](https://docs.docker.com/registry/recipes/nginx/)/[apache](https://docs.docker.com/registry/recipes/apache/), возможно если сделать еще один подход, то будет счастье

## Заключение {#resume}

 - экстремально уменьшили время разворачивания окружений, стало легко перемещаться “во времени”  между ними;
 - разгрузили центральные серверы, задействовали простаивающие локальные мощности компьютеров разработчиков;
 - автоматизированные повторяемые сборки SDK/окружений;
 - ввели в строй CI по основным проектам и по сборкам самих окружений;
 - в общем уменьшились издержки - все довольны;

Теперь алгоритм входа нового сотрудника в проект выглядит так:
 - git clone `<repo_url>` && cd project_dir
 - ./dockerBuild.sh platformName mode path_to_project optional_cmd
 - запуск/деплой приложения/сервиса;
 - ???????
 - PROFIT!


Презенташка от доклада для [С++ CoreHard Spring 2018 ](http://conference.corehard.by/) conference:
<iframe width="720" height="405" src="https://docs.google.com/presentation/d/1i_l1pz-J5b6hGABpa1C5a9kxagLW3GU20m1xrQ3zQ2g/embed" frameborder="0"></iframe>
