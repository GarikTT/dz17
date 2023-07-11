Урок 17 - SELinux - когда все запрещено.
Цель занятия - 	объяснить, что такое системы принудительного контроля доступа;
                объяснить, как работает система SELinux;
                работать с системой SELinux.
Результат занятия - работать с системой SELinux, не отключая ее и получение навыков конфигурирования политик SELinux.
Тренировка умения работать с SELinux: диагностировать проблемы и модифицировать политики SELinux для корректной работы приложений, если это кому-нибудь требуется.
Домашнее задание - 
    1. Запустить nginx на нестандартном порту 3-мя разными способами:
        1.1 Переключатели setsebool;
        1.2 Добавление нестандартного порта в имеющийся тип;
        1.3 Формирование и установка модуля SELinux.

script --timing=time_loading_lesson_one_log loading_lesson_one.log
1. Запускаем предложенные в ДЗ нам с вами вложенные две виртуальные машины - vagrant up.
   Результат - вот, прям, как в методичке, те же самые ошибки.
2. Заходим на сервер, переходим в root. Эти команды не буду описывать.
3. Проверяем состояние файервола - systemctl status firewalld
4. Проверяем конфигурацию nginx - nginx -t
5. Проверяем режим работы SELinux - getenforce
6. "Далее разрешаем в SELinux работу nginx на порту TCP 4881 c помощью переключателей setsebool."
    Вот здесь начинаю просыпаться и включать свой моск - "Находим в логах (/var/log/audit/audit.log) информацию о блокировании порта".
    Ага, мы с вами знаем порт - 4881! С вероятностью 99 из девносто девяти возможных это будет одна строка.
    Выполняем - cat /var/log/audit/audit.log | grep 4881 - 
    type=AVC msg=audit(1688763210.407:832): avc:  denied  { name_bind } for  pid=2855 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unre
    served_port_t:s0 tclass=tcp_socket permissive=0
7. "Копируем время, в которое был записан этот лог, и, с помощью утилиты audit2why смотрим grep 1688763210.407:832 /var/log/audit/audit.log | audit2why" - не работает!!!
    Судя по названию команды - это анализатор логов (перевод - почему это случилось?). С помощью софтГугления находим решение.
7. Устанавливаем audit2why - yum install policycoreutils-python
8. Снова выполняем предыдущую команду - grep 1688763210.407:832 /var/log/audit/audit.log | audit2why. Отлично! Заработало!!!
9. Читаем -
	    Was caused by:
        The boolean nis_enabled was set incorrectly.
        Description:
        Allow nis to enabled

        Allow access by executing:
        # setsebool -P nis_enabled 1
10. Переводим в уме - "Подсказка: nis нужно энейблить". И вот, в этот самый момент все Человечество (для меня) сразу разделилось на 10 типов людей - тех, 
    кто понимает двоичную систему счисления и тех, кому это никогда не пригодится в зтой счастливой жизни.
    Что ставить? 1, on, true, enable, наверное да??? The boolean nis_enabled was set incorrectly - а что там щас установлено? 
    Это значение по дефолту не правильное?! Там мнимая единица, что ли?
9. Включаем параметр nis_enabled и перезапускаем nginx - 
    setsebool -P nis_enabled on
    systemctl restart nginx
    systemctl status nginx
10. Далее по методичке все отработало на ура -
    getsebool -a | grep nis_enabled
    setsebool -P nis_enabled off
    semanage port -l | grep http
    semanage port -a -t http_port_t -p tcp 4881
    semanage port -l | grep  http_port_t
    systemctl restart nginx
    systemctl status nginx
    http://127.0.0.1:4881 (вот здесь пришлось устанавливать Яндекс, т.к. Firefox грузит всю машину, включая виртуальные, по самое досвиданье минут на десять)
    semanage port -d -t http_port_t -p tcp 4881
    semanage port -l | grep  http_port_t
    systemctl restart nginx
    systemctl status nginx
    grep nginx /var/log/audit/audit.log
    grep nginx /var/log/audit/audit.log | audit2allow -M nginx
    semodule -i nginx.pp
    systemctl start nginx
    systemctl status nginx
    semodule -l
    semodule -r nginx - удаляем.
11. scriptreplay --timing=time_loading_task_one_log loading_task_one.log -d 20
12. Первое задание сделано.

13. Второе задание - Обеспечение работоспособности приложения (видимо любого) при включенном SELinux.
    Насколько я понял задание - необходимо внести изменения в настройки DNS сервера ns01.
    Запускаем script --timing=time_loading_task_two_log loading_task_two.log
14. Выполним клонирование репозитория: git clone https://github.com/mbfx/otus-linux-adm.git
    Уфф, хоть сдеся никаких косяков не произошло. А я уже чувствую, что дальше этот квест будет еще интереснее.
15. Переходим к упражнению - 
    cd otus-linux-adm/selinux_dns_problems
16. vagrant up
    "Здесь зрители аплодирут, аплодируют..." и вылетает VirtualBox. Перезагружаемся.
    vagrant status
    vagrant ssh client
17. Сразу проверяем правила в настройках произвольных DNS серверов -
    dig @192.168.50.10 ns01.ddns.lab
    dig @192.168.50.10 alfatelplus.ru
    dig @192.168.50.10 google.com
    dig @192.168.50.10 ns01.dns.lab
    dig @192.168.50.10 ns01.ss.lab
    Вот здесь даже корневой сервер Интернета повесился. Ну нету такого адреса, он свободен!!!
    Вы его можете купить за условные американские рубли и использовать для рассылки спама.
    Я, еще по молодости, когда делал свой DNS-сервер на MS Windows'е 2003 промучился меньше,
    чем при выполнении этого задания и написания README.md!!!
18. Проверяем правила на клиенте -
    dig @192.168.50.15 client.ddns.lab
    Нету!
19. Настраиваем -
    nsupdate -k /etc/named.zonetransfer.key
    Устанавливаем правило для ddns.lab -
    server 192.168.50.10
    zone ddns.lab
    send
    Косяк, как и предвиделось, если следовать нашей с вами методичке.
    quit
20. sudo -i
    cat /var/log/audit/audit.log | audit2why
    Тут мы видим, что на клиенте отсутствуют ошибки.
21. Через другой wcm подключаемся -
    vagrant ssh ns01
    sudo -i
    cat /var/log/audit/audit.log | audit2why
    В логах мы видим, что ошибка в контексте безопасности. Вместо типа named_t используется тип etc_t.
    Проверим данную проблему в каталоге /etc/named:
    ls -laZ /etc/named
    sudo semanage fcontext -l | grep named
    Изменим тип контекста безопасности -
    sudo chcon -R -t named_zone_t /etc/named
    Проверяем - ls -laZ /etc/named
    Контекст поменялся!
    Выходим из сервера -
    exit
    exit
22. Заходим в клиента - 
    vagrant ssh client
    Снова пытаемся изменить правила -
    nsupdate -k /etc/named.zonetransfer.key
    show
    server 192.168.50.10
    zone ddns.lab
    update add www.ddns.lab. 60 A 192.168.50.15
    send
    Ничего не изменилось.
23. Переходим на сервер - 
    Выполняем волшебную команду sudo setenforce 0
24. Опять на клиенте проводим все манипуляции - 
    dig @192.168.50.10 ns01.ddns.lab
    dig @192.168.50.10 alfatelplus.ru
    dig @192.168.50.10 google.com
    dig @192.168.50.10 ns01.dns.lab
    dig @192.168.50.10 ns01.ss.lab
    Ок!!!
25. Перезагружаемся и проверяем на клиенте - dig @192.168.50.10 www.ddns.lab
26  scriptreplay --timing=time_loading_task_two_log loading_task_two.log -d 20
27. Все!
28. Каталог otus-linux-adm непонятно загрузился или нет?!
    В нем я не делал никаких изменений, открывается по ссылке https://github.com/mbfx/otus-linux-adm.git
29. А эта команда не вошла в записанный сценарий, я ее проверил для надежности:
[vagrant@client ~]$ dig www.ddns.lab
; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.13 <<>> www.ddns.lab                                                                     
;; global options: +cmd                                                                                                                 
;; Got answer:                                                                                                                          
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 1333                                                                                                                     
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2                                                                 
                                     
;; OPT PSEUDOSECTION:                                                                                                                                                        
; EDNS: version: 0, flags:; udp: 4096                                                                                                                                        
;; QUESTION SECTION:                                                                                                                                                         
;www.ddns.lab.                  IN      A                                                                                               
                                     
;; ANSWER SECTION:                                                                                                                                                           
www.ddns.lab.           60      IN      A       192.168.50.15                                                                           
                                     
;; AUTHORITY SECTION:                                                                                                                                                        
ddns.lab.               3600    IN      NS      ns01.dns.lab.                                                                           
                                     
;; ADDITIONAL SECTION:                                                                                                                                                       
ns01.dns.lab.           3600    IN      A       192.168.50.10                                                                           
                                     
;; Query time: 6 msec                                                                                                                                                        
;; SERVER: 192.168.50.10#53(192.168.50.10)                                                                                                                                   
;; WHEN: Mon Jul 10 03:04:23 UTC 2023                                                                                                                                        
;; MSG SIZE  rcvd: 96                                                                                                                   
[vagrant@client ~]$  
