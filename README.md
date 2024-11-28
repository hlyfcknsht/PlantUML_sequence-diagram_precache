# PlantUML_sequence-diagram_precache
Предварительное кэширование данных интеграционным скриптом


1. Клиент направляет реестр данных
2. Интеграционный скрипт отправляет данные в коллекцию интеграционной базы
3. Данные записаны
4. По методу build_id запрашиваем данные билда
5. Данные получены
6. Парсим полученные данные на месседжи, опорные аудио, голос диктора и tts settings
7. Интеграция направляет запрос на прекэш данных:
  7.1. registry_id
  7.2. Поля данных объекта
  7.3. Настройки билда
  7.4. Ссылка на месседжи и опорные аудио
  7.5.Имя интеграционной коллекции
8. Получение данных из запроса
9. Передача полей в сервис прекэша
10. Цикл передачи данных для кеширования фраз
11. Каждые 100 записей передаём отклик в очередь рэббита
12. Получение отклика о прогрессе предкэша
13. Когда все данные закэшились отправляем отклик о завершении
14. Очередь удаляется после окончания предкэша


@startuml
autonumber
/'СТРУКТУРА СХЕМЫ'/
actor Клиент as client
box Intergation API
  participant "Integration script" as participant.integrator
  database "Интеграционная база" as idb
box end
box System API
  participant "Портальное апи" as portal.api
box end
box Precache
  participant "Сервис прекэша" as precache
  participant "tts proxy" as tts.proxy
box end
box Precache queue
  queue kv.tts.precaching.v2 as queue.precahe
  queue project.tts.precache.uid as precache.result
box end

/'ВЗАИМОДЕЙСТВИЕ СХЕМЫ'/

==Запись данных в интеграционную базу==

client -> participant.integrator: Реестр данных
participant.integrator -> idb: Данные
activate idb
participant.integrator <<-- idb: Данные записаны
deactivate idb
participant.integrator -> portal.api: Метод build_id, Получаем данные билда
activate portal.api
portal.api -->> participant.integrator: Данные билда
deactivate portal.api
activate participant.integrator
participant.integrator <- participant.integrator:  Парсинг Месседжи, опорные аудио, speaker id, TTS settings
deactivate portal.api

==Запрос на предкеширование==

participant.integrator -> queue.precahe: Запрос на предкэш данных
deactivate participant.integrator
note right
  registry_id
  Поля данных (obj)
  Настройки билда {
  PRECACHE_VOICE - голос диктора (Связать со speaker id)
  PRECACHE_SPEED - скорость 
  PRECACHE_MODEL_SERVER - модель озвучки
  PRECACHE_PRECACHE_QUEUE - очередь, в которой слушаем ответ
  PRECACHE_VOLUME_LEVEL - громкость голоса
  PRECACHE_SAMPLE_RATE - Частота дискретизации
  PRECACHE_CALL_CENTRE_NOISE - шум коллцентра
  }
  [{"template": "шаблон", "audio": "ссылка", "value_from": "путь до значения филдов"}, ...]
  имя интеграционной коллекции
end note
activate queue.precahe
idb <- queue.precahe: Запрос данных по registry_id(PK)
deactivate queue.precahe
activate idb

==Предкэш==

precache <<-- idb: Поля для предкеширования
deactivate idb
activate precache
loop Кол-во повторений == количеству кешируемых фраз
  precache -> tts.proxy: Кэширование данных
  deactivate precache
  activate tts.proxy
end

==Результат предкэша==

loop раз в 100 записей
  tts.proxy -->> precache.result: Прогресс кеширования
  deactivate precache
  activate precache.result
  deactivate tts.proxy
  precache.result -->> participant.integrator: Прогресс предкэша 
end loop
group Закончились записи в очереди
  precache.result -->> participant.integrator: Все данные предкэшированы
  deactivate precache.result
end
deactivate precache.result
precache.result -[#red]> precache.result: Удаление очереди
@enduml
