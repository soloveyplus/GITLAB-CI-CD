### Локальная настройка GITLAB + gitlab runner + docker registry для CICD
Так как работа предполагается в локальном пользовании, сертификаты не используем.\
Так же и для docker registry. Все по HTTP.
1. Создаем записи типа A – gitlab и registry
2. GITLAB устанавливается на дистрибутив ubuntu server 20.04
3. В  `/etc/gitlab/gitlab.rb`  меняем -  `external_url 'http://gitlab.dima.local'`
4. Реконфигурируем наш гитлаб -  `gitlab-ctl reconfigure`
5. Пароль админа смотрим - `/etc/gitlab/initial_root_password`
6. На данном этапе гитлаб работает. 
7. Устанавливаем связку – docker, docker-compose
8. Для целей CI/CD настраиваем раннеры.\
Создаем том для Docker:\
`docker volume create gitlab-runner-config` ,
где `gitlab-runner-config` – название тома \
Запускаем контейнер GitLab Runner, используя только что созданный том:

    >docker run -d --name gitlab-runner --restart always \\    
    -v /var/run/docker.sock:/var/run/docker.sock \\\
    -v gitlab-runner-config:/etc/gitlab-runner \\\
    gitlab/gitlab-runner:latest
   
В нашем проекте гитлаба в меню-Settings-CI/CD-Runners находим информацию для добавления нового раннера – адрес и токен. Копи-пастим\
Заходим во внутрь контейнера раннера- \
`docker exec -it <наш_раннер> gitlab-runner register` , где `gitlab-runner register` – команда регистрации\
В ответ копи-пастим адрес и токен из проекта. Далее описание – можно по дефолту. \
По поводу тэгов – обязательно нужны – данной опцией можно управлять какие раннеры должны выполняться. Так как собираем контейнеры то ставим tag – docker. \
Далее выбираем экзикьютеры – выбираем – docker.\
Спросит какой использовать контейнер \
для сборок - `docker:20.10.22-dind` .  Это образ докер в докере. Именно   20.10.22 т.к. последняя имеет проблемы.\
Проверяем в меню нашего проекта появление нашего docker раннера.\
В процессе сборки может появится ошибка типа – ERROR: during connect…..\
лечится так – \
Смотрим подробную инфу о нашем созданном томе –\
`docker inspect gitlab-runner-config` , где `gitlab-runner-config` - наш том\
Находим, что он находится - `/var/lib/docker/volumes/gitlab-runner-config/_data`\
Меняем каталог на -  `/var/lib/docker/volumes/gitlab-runner-config/_data`\
Редактируем -  `config.toml`  
Находим наш раннер и в строке – Volumes - в квадратных скобках через запятую к записи рядом – `“/cache” добавляем – “/var/run/docker.sock:/var/run/docker.sock”`\
Сохраняем и перезапускаем docker.\
9. Настраиваем для нашего проекта – Container Registry\
Он будет держаться гитлабом. И авторизация то же гитлабом. \
Для этого в - `/etc/gitlab/gitlab.rb`
раскомитим и дабавим:\
`registry_external_url 'http://registry.dima.local:5050'`   - работаем по http\
`gitlab_rails['registry_enabled'] = true`    -  включаем внутренний репозитарий гитлаба\
_________не проверял__________\
`registry['registry_http_addr'] = "127.0.0.1:5000"`\
`registry['debug_addr'] = "localhost:5001"`\
_______не проверял________________ \
Далее реконфигурируем наш гитлаб -  `gitlab-ctl reconfigure`\
Но это еще не все.\
Отключаем https в  docker – для этого создаем или редактируем - `/etc/docker/daemon.json` с содержимым: 

>{\
  "insecure-registries" : ["registry.dima.local:5050"]\
} 



Сохраняем и перезапускаем docker.\
Проверяем –    `docker login  registry.dima.local:5050`\
Пароль и логин от нашего гитлаба.  Login Succeeded\
Теперь все должно быть настроено!!!!


**И последнее касаемо файла - `.gitlab-ci.yml` и тэгов**\
Что бы заработал наш раннер вначале добавим:

>default:\
 &nbsp;&nbsp;&nbsp;&nbsp;tags:\
    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- docker





 


