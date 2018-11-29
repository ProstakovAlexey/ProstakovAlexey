Установка драйвера ODBC в Linux
===============================

Установка драйвера для MS SQL от Microsoft
------------------------------------------
Пока этот вариант выглядит наилучшим. Я скачал драйвер со станицы https://docs.microsoft.com/ru-ru/sql/connect/odbc/download-odbc-driver-for-sql-server.
Установил последнюю версию драйвера для Ubuntu 16.04 - https://packages.microsoft.com/ubuntu/16.04/prod/pool/main/m/msodbcsql/msodbcsql_13.1.9.1-1_amd64.deb
Процесс установки:

    1. Скачиваю драйвер.
    2. Перейти в папку с драйвером и выполнить команду ``sudo dpkg -i msodbcsql_13.1.9.1-1_amd64.deb``
    3. После установки выполнить проверку ``odbcinst -d -q``, она выдает список драйверов, в нем появилась строка ``[ODBC Driver 13 for SQL Server]``

После установки драйвер можно использовать в программе, пример в конце этой статьи.


Установка драйвера FreeTDS
--------------------------

.. warning:: При работе с FreeTDS в Linux 16.04 есть проблема с кодировкой, похоже он не воспринимает utf-8. Так же есть проблема с ``pypyobdc`` https://github.com/jiangwen365/pypyodbc/issues/68

Драйвер использую для соединения с MS SQL Server, проверял работу для 2008r2 и 2012r2.

Установка необходимых пакетов 
*****************************
Установка и настройка проводилась на Linux Ubuntu 12.04.1 и на Ubuntu 14.04 64bit:       
``sudo apt-get install tdsodbc unixodbc unixodbc-bin unixodbc-dev odbcinst freetds-bin``

Настройка FreeTDS
*****************
Редактировать настроечный файл ``sudo gvim /etc/freetds/freetds.conf``
У меня настроечный файл выглядит так (убрал комментарии, чтобы экономить место)::

        [global]
        ;	tds version = 4.2
        ;	dump file = /tmp/freetds.log
        ;	debug flags = 0xffff
        ;	timeout = 10
        ;	connect timeout = 10
	        text size = 64512
                client charset = cp1251 
        #  я добавил вот эту секцию, отсюда и до конца
        # у меня именованный экземпляр, поэтому нестандартный порт
        [FreeTDS_2012r2]
	host = 192.168.0.2
  	port = 49223
  	tds version = 8.0
  	client charset = UTF8
        
Теперь необходимо выполнить проверку TDS, чтобы было соединение с БД и корректно отображадись русские символы (eсли у вас не оказалось программы tsql, то можно найти в каком она пакете : ``apt-file search tsql`` (в Ubuntu 12.04 она в пакете freetds-bin) и установить)::

        $ tsql -S FreeTDS_2012r2 -U sa -P 111
        locale is "ru_RU.UTF-8"
        locale charset is "UTF-8"
        using default charset "UTF8"
        1> select top 10 * from dotProject2.dbo.obr
        2> go

Настройка ODBC
**************
Определяем где хранятся настроечные файлы::
        
        $  odbcinst -j
        unixODBC 2.3.1
        DRIVERS............: /etc/odbcinst.ini
        SYSTEM DATA SOURCES: /etc/odbc.ini
        FILE DATA SOURCES..: /etc/ODBCDataSources
        USER DATA SOURCES..: /home/alexey/.odbc.ini
        SQLULEN Size.......: 8
        SQLLEN Size........: 8
        SQLSETPOSIROW Size.: 8

Нам необходимо будет редактировать файлы **drivers** и **system data source**. В файлу **drivers** необходимо создать секцию для FreeTDS и указать место положения драйверов: ``sudo gvim /etc/odbcinst.ini``
У меня это выглядит так, если вы не знаете где находятся ваши драйверы, то их можно найти командой ``find / -name libtds*``::

        [FreeTDS]
        Description = TDS driver (Sybase/MS SQL)
        # Some installations may differ in the paths
        Driver = /usr/lib/x86_64-linux-gnu/odbc/libtdsodbc.so
        Setup = /usr/lib/x86_64-linux-gnu/odbc/libtdsS.so
        FileUsage = 1
        UsageCount = 4

Устанавливаем параметры драйвера в систему: ``sudo odbcinst -i -d -f /etc/odbcinst.ini``

Проверяем установленный драйвер::

        $ odbcinst -d -q
        [FreeTDS]

Чтобы удалить параметры драйвера из системы: ``sudo odbcinst -u -d -n FreeTDS``

Ссылки
******
  - Элемент нумерованного спискаConnecting to a Microsoft SQL Server database from Python under Ubuntu http://www.tryolabs.com/Blog/2012/06/25/connecting-sql-server-database-python-under-ubuntu/
  - Настройка доступа к Microsoft SQL Server через ODBC. Ubuntu 12.04 http://alah-my.blogspot.ru/2013/02/microsoft-sql-server-odbc-ubuntu-1204.html
  - Обсуждение на форуме http://python.su/forum/topic/20820/

Пример программы
----------------
Это пример, на котором я тестировал. С драйвером от microsoft она работает в полном объеме. Для freetds не работает передача строкой, какая-то проблема с кодировками::

    # -*- coding: utf-8 -*-
    import pyodbc
    con_str = "DRIVER={ODBC Driver 13 for SQL Server}; SERVER=192.168.0.2,49223; DATABASE=dotProject2; UID=sa; PWD=111;"
    print('pyodbc version:', pyodbc.version)
    con = pyodbc.connect(con_str)
    cur = con.cursor()
    sql_str = r"SELECT id FROM Tasks WHERE name='Подготовка описания сервиса' AND TopicId=193"
    cur.execute(sql_str)
    print('1) bad select type A by sql string:', cur.fetchone())
    cur.execute('SELECT id FROM Tasks WHERE name=? AND TopicId=?', ('Подготовка описания сервиса', 193))
    print('2) good select type A by sql param:', cur.fetchone())
    cur.execute('SELECT id FROM Tasks WHERE name=? AND TopicId=?', ('Изменение в руководство пользователя АСП', 193))
    print('3) good select type B by sql param:', cur.fetchone())
    cur.execute('SELECT id FROM Tasks WHERE name=? AND TopicId=?', ('Изменение в руководство пользователя и хвост', 193))
    cur.execute('SELECT id FROM Tasks WHERE name=? AND TopicId=?', ('Изменение в руководство пользователя АСП', 193))
    print('4) bad select type B by sql param:', cur.fetchone())
    con.close()

Обсуждение проблем - https://vk.com/python_community?w=wall-38080744_60013

