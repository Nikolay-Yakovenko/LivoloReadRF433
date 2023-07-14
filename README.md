# LivoloReadRF433 for Home Assistant
Чтение кодов **RF433** брелоков от фирмы **Livolo** с помощью **ESPHome**.

Reading codes **RF433** key fobs from **Livolo** using **ESPHome**.

## О проекте

**Задача:** Использование пульта Livolo работающего на частоте 433МГц,
 в автоматизациях умного дома Home Assistant. Для работы используется ESPHome.

**Используемые компоненты:**
- ESP8266-01S 

<img src="photo\ESP8266-01S.jpg"/>

- Приемник на 433МГц QIACHIP RX500A

<img src="photo\QIACHIP RX500A top1.jpg" width="320"/>

<!-- - Питание на Ваш вкус. Я использую Hi-Link HLP-PM01 (5В, 0.6А) -->
- Питание на Ваш вкус. Я использую Hi-Link HLP-PM03 (3.3В, 3W)
<img src="photo\HLK-PM03_AC-DC.jpg" width="320"/> 
<!--// height="300" />-->

- Пульт Livolo (обозначение на плате VL-RMT02 VER:B6)

<img src="photo\photo_1.jpg" width="320"/> 

<!--<img src="photo\photo_2.jpg" width="320"/> -->
<img src="photo\photo_3.jpg" width="320"/> 

<img src="photo\photo_4.jpg" width="320"/> 



 
## Подключение:

Один из контактов приемника **(DO)** подключается к контакту **GPIO2** ESP8266-01S.

<img src="photo\QIACHIP RX500A 433MGz top.jpg" width="301"/> 
<img src="photo\ESP8266-01S-pinout.jpg" width="320"/> 


## Прошивка:

- В ESPHome создаем новое устройство, называем (например) **livoloreadrf433**.
Прошиваем его стартовой прошивкой.
Затем кладем файлик **[LivoloReadRF433.h](/config/esphome/LivoloReadRF433.h)** по пути **"config\esphome\\"**
рядом с **[livoloreadrf433.yaml](/config/esphome/livoloreadrf433.yaml)**, который будет создан в системе.

- Редактируем файл настроек **livoloreadrf433.yaml**
  - прописываем в начале файла:

    ```yaml
    esphome:
      includes:
        - LivoloReadRF433.h

    ```

  - и добавляем сенсоры:

    ```yaml
    sensor:
    - platform: custom
      lambda: |-
        auto my_sensor = new LivoloReadRF433();
        App.register_component(my_sensor);
        return {my_sensor->RemoteID ,my_sensor->KeyCode};

      sensors:
        - name: "RemoteID"
          force_update: true
        - name: "KeyCode"    
          force_update: true
    ```

    В результате в Home Assistant, в интеграции ESPHome появится устройство LivoloReadRF433, с двумя сенсорами RemoteID (код пульта) и KeyCode (код кнопки).

## Пример автоматизации:

```yaml
alias: >-
  Большая комната. Свет. Переключение одной лампочки по кнопке "А" пульта Livolo 
description: >-
  Переключаем одну лампочку в большой комнате при нажатии на кнопку "А" пультика Livolo
trigger:
  - platform: state
    entity_id:
      - sensor.livoloreadrf433_keycode  #такой сенсор будет создан, если имя устройства задали livoloreadrf433
    from: null
    to: "8" #код кнопки
condition:
  - condition: state
    entity_id: sensor.livoloreadrf433_remoteid #такой сенсор будет создан, если имя устройства задали livoloreadrf433
    state: "32375" #код пульта
action:
  - service: switch.toggle  #запускаем службу switch для переключения света
    data: {}
    target:
      entity_id:
        - switch.YOU_SWITCH_ENTITY_ID 
mode: single
```
где YOU_SWITCH_ENTITY_ID сущность вашего выключателя.

## Дополнение:
    Уже не актуально !!!
    Был изменен код и на данный момент все работает нормально. 
    После передачи кодов сделано принудительное обнуление значений сенсоров.

При тестировании обнаружилась проблема: при нажатии на одну и туже кнопку пульта, в Home Assistant передавалось только первое нажатие. Если нажимать разные кнопки - всё передавалось нормально.

Т.е. в логе ESPHome устройства был виден перехват кода кнопки, но т.к. его значение не изменялось, передача в HA не осуществлялась.

Для исправления данной ситуации был использован параметр "force_update: true", который должен был заставить ESPHome передавать данные, даже если код клавиши не изменился.

Но, выяснилось, что параметр не отрабатывает как задумывался (по крайней мере у меня). [Почитать об этом можно сдесь.](https://github.com/home-assistant/core/issues/91221)

В результате пришлось добавить принудительное изменение значения 
сенсора **keycode** в автоматизации Home Assistant (записываем, значение, которое заведомо не совпадает с любым кодом клавиш).

Для включения возможности изменения значений датчиков через автоматизации, надо:
1. В configuration.yaml добавить "python_script:"
2. В папке config (в ней лежит configuration.yaml) создать папку "python_scripts"
3. В папку "python_scripts" положить файл "[set_state.py](/config/python_scripts/set_state.py)"
4. Перезагрузить HA.

Более подробно описано в [статье](https://kvvhost.ru/2020/05/03/python-set-state-ha/) (статья не моя).

Код обработчика нажатий клавиш пульта взят [здесь](https://forum.arduino.cc/t/control-livolo-switches-livolo-switch-library/149850).

## Благодарности:

 - [Andrew](https://github.com/andrewjswan) за советы и помощь с кодом.
 - [Телеграм-канал посвященный ESPHome](https://t.me/esphome)
 - [Сайт ESPHome](https://esphome.io)